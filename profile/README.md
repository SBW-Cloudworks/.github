![sbwCloudworks](swbCloudworksBanner.png)

## Technical Architecture Diagram
![ClickStreamDiagramV8](ClickStreamDiagramV8.png)
# 📊 Clickstream Analytics Platform for E-Commerce  
Batch-based ETL • AWS Serverless • Data Warehouse • R Shiny Analytics

---

## 🏆 Overview

This project implements a **Batch-Based Clickstream Analytics Platform** for an e-commerce website selling computer products.

The system collects clickstream events from the frontend, stores raw JSON data in **Amazon S3**, processes events via scheduled ETL (AWS Lambda + EventBridge), and loads analytical data into a dedicated **PostgreSQL Data Warehouse** on EC2.

Analytics dashboards are built using **R Shiny**, deployed in a private subnet and directly querying the Data Warehouse.

The platform is engineered with:

- Clear separation between **OLTP vs Analytics** workloads  
- Private-only analytical backend (no public DW access)  
- Cost-efficient, scalable AWS serverless components  
- Minimal moving parts for reliability and simplicity  

---

# 🏗️ Architecture Summary

## 1. User-Facing Domain

### Frontend

- Built using **Next.js + React**
- Hosted on **AWS Amplify Hosting**
- Amplify internally leverages:
  - **Amazon CloudFront** (global CDN)
  - **Amazon S3** (static assets bucket)
- Authentication handled by:
  - **Amazon Cognito User Pool**

### Operational Database (OLTP)

- A standalone **EC2** instance running **PostgreSQL**
- Stores:
  - Users
  - Products
  - Orders
  - OrderItems
  - Inventory & transactional data
- Located in the **Public Subnet** so that Amplify’s SSR / API routes can connect via **Prisma** using `DATABASE_URL`

> Note: In a strict production design OLTP would typically be private (e.g. RDS in private subnet),  
> but this architecture intentionally allows public OLTP EC2 so that Amplify (which is not inside the VPC) can connect directly.

---

## 2. Ingestion & Data Lake Domain

### Ingestion Flow

1. The frontend records user behavior (page views, clicks, interactions).
2. Events are POSTed as JSON to **Amazon API Gateway (HTTP API)** at:
   ```http
   POST /clickstream
   ```
3. API Gateway invokes a **Lambda Ingest Function**.
4. Lambda Ingest:

   * Validates the payload
   * Enriches metadata (timestamps, user/session IDs, etc.)
   * Writes raw JSON into the **S3 Raw Clickstream Bucket**:

   ```text
   s3://<raw-clickstream-bucket>/events/YYYY/MM/DD/HH/events-<uuid>.json
   ```

### Batch ETL Flow

* **Amazon EventBridge** defines a **cron rule** (e.g. every 30 minutes).
* On each schedule:

  * EventBridge triggers **Lambda ETL** (configured inside the VPC).
  * Lambda ETL:

    * Reads the new raw files from **S3 Raw Bucket**
    * Cleans, normalizes, and sessionizes events
    * Converts NoSQL-style JSON into **SQL-ready analytic tables**
    * Inserts processed data into the **PostgreSQL Data Warehouse** hosted on EC2 in a private subnet

No additional “processed” S3 bucket is used — processed data is written directly to SQL tables in the DW.

---

## 3. Analytics & Data Warehouse Domain

The analytics environment uses **two EC2 instances**, each with a dedicated role.

### EC2 #1 — OLTP Database (Public Subnet)

* PostgreSQL database for the e-commerce application
* Serves live operational traffic:

  * Product listing
  * Cart/checkout
  * Orders, inventory, users
* Accessible over the internet only to:

  * Amplify SSR / backend
  * Admin / maintenance IPs (via Security Groups)

---

### EC2 #2 — Data Warehouse + R Shiny (Private Subnet)

#### PostgreSQL Data Warehouse

* Stores curated clickstream analytics schema:

  * event_id
  * event_timestamp
  * event_name
  * user_id
  * user_login_state
  * identity_source
  * client_id
  * session_id
  * is_first_visit
  * product_id
  * product_name
  * product_category
  * product_brand
  * product_price
  * product_discount_price
  * product_url_path
  * Aggregated metrics tables
* Located in a **Private Subnet** (no public IP)
* Receives data exclusively from **Lambda ETL** within the VPC

#### R Shiny Analytics Server

* Runs on the same EC2 instance as the DW
* Connects locally to the DW database
* Hosts interactive dashboards visualizing:

  * User journeys
  * Conversion funnels
  * Product engagement
  * Time-based activity trends

> OLTP and Analytics are fully separated, ensuring reporting queries do not impact transactional performance.

---

# 🔐 Networking & Security Design

## VPC Layout

* **VPC CIDR**: `10.0.0.0/16`
* **Subnets**:

  * **Public Subnet (OLTP)**

    * EC2 PostgreSQL OLTP
    * Internet Gateway for public connectivity
  * **Private Subnet 1 (Analytics)**

    * EC2 Data Warehouse (PostgreSQL)
    * EC2 R Shiny Server
  * **Private Subnet 2 (ETL Layer)**

    * Lambda ETL (VPC-enabled)
    * S3 Gateway Endpoint

## Routing

* **Public Route Table**

  * `0.0.0.0/0` → Internet Gateway
* **Private Route Tables**

  * Local VPC routes
  * Prefix list routes for S3 via **Gateway VPC Endpoint (S3)**

No NAT Gateway is used — private components reach S3 using the S3 VPC Endpoint.

## Security Groups

* **SG-OLTP**

  * Inbound:

    * `5432/tcp` – from Amplify / trusted IPs (for Prisma)
    * `22/tcp` – from admin IP (for SSH)
  * Outbound: default (all allowed)

* **SG-DW**

  * Inbound:

    * `5432/tcp` – from Lambda ETL SG and Shiny SG
  * Outbound: default (all allowed)

* **SG-Shiny**

  * Inbound: restricted to admin/VPN only (or internal admin tools)
  * Outbound: permitted to DW (localhost or private IP)

* **SG-ETL-Lambda**

  * No inbound (Lambda does not accept inbound)
  * Outbound: allowed to S3 endpoint + DW SG via private networking

## IAM & Monitoring

* Dedicated IAM roles per Lambda function:

  * **Lambda Ingest Role**: S3 write-only (Raw bucket)
  * **Lambda ETL Role**: S3 read + DB access permissions
* **CloudWatch Logs** for:

  * API Gateway access logs
  * Lambda Ingest & ETL logs
  * ETL execution metrics

---

# 📦 S3 Buckets (2 Buckets Only)

1. **Amplify Assets Bucket**

   * Stores static website assets (JS, CSS, images, etc.)
   * Managed by Amplify Hosting

2. **Raw Clickstream Data Bucket**

   * Stores raw JSON clickstream events from Lambda Ingest
   * Partitioned by date/hour to support batch ETL

No additional “processed” bucket is required; all processed data is loaded directly into the PostgreSQL Data Warehouse.

---

# 🔁 Data Flow Summary

1. User accesses the web app via **CloudFront → Amplify**
2. User authenticates via **Cognito** and interacts with the UI
3. Frontend sends clickstream events to **API Gateway**
4. **API Gateway → Lambda Ingest**
5. Lambda Ingest writes event JSON files into **S3 Raw Clickstream Bucket**
6. **EventBridge Cron** (every 30 minutes) triggers **Lambda ETL**
7. Lambda ETL:

   * Reads partitioned raw files from S3
   * Cleans and transforms JSON into SQL-ready data
8. Lambda ETL connects to **EC2 Data Warehouse** in the private subnet
9. ETL inserts processed rows into DW tables (sessions, events, funnels, etc.)
10. **R Shiny** reads from DW and renders analytics dashboards
11. Admin views dashboards (via secure/private access) for insights

---

# 🧩 Key Features

* Batch clickstream ingestion using API Gateway + Lambda + S3
* Serverless ETL with EventBridge scheduling
* Clear separation between:

  * **OLTP** (online transaction processing)
  * **Analytics / Data Warehouse**
* R Shiny-based visual analytics, fully private
* Cost-optimized:

  * No NAT Gateway
  * S3 for raw storage
  * Lambda-based compute for ETL
* Direct PostgreSQL connectivity from Amplify using Prisma

---

# 🛠️ Tech Stack

### AWS Services

* **AWS Amplify Hosting** — Next.js hosting (SSR + static assets)
* **Amazon CloudFront** — CDN edge distribution
* **Amazon Cognito** — User authentication and identity
* **Amazon S3** — Static assets + raw clickstream data
* **Amazon API Gateway (HTTP API)** — Ingestion endpoint for events
* **AWS Lambda (Ingest & ETL)** — Serverless compute for data pipeline
* **Amazon EventBridge** — Scheduled ETL triggers (cron job)
* **Amazon EC2** — OLTP DB + DW + Shiny
* **Amazon VPC** — Network isolation (public & private subnets)
* **AWS IAM** — Access control
* **Amazon CloudWatch** — Logging & monitoring

### Databases

* **PostgreSQL (EC2 OLTP)** — Operational database for the e-commerce app
* **PostgreSQL (EC2 Data Warehouse)** — Analytical database for clickstream data

### Analytics

* **R Shiny Server** — Analytics dashboards
* **Custom ETL logic** — Lambda ETL transforming S3 JSON → SQL tables

---

# 🚀 Deployment Notes

* **No NAT Gateway** is required (S3 access via VPC Gateway Endpoint)
* All analytical components (DW + Shiny + ETL Lambda) sit in **private subnets**
* Only the OLTP EC2 instance is public, to support direct Prisma connections from Amplify
* For a production-hardening step, OLTP could be migrated to:

  * **Amazon RDS PostgreSQL in private subnets**
  * Combined with a dedicated backend API layer

---

# 📚 Local Development (LocalStack Notes)

When using LocalStack, additional internal/system buckets may be created automatically.
However, the project **logically depends on only two S3 buckets**:

* Amplify Assets Bucket
* Raw Clickstream Bucket

Some services such as Amplify Hosting, Cognito UI flows, and full VPC networking may be only partially supported in LocalStack and require integration testing in real AWS.

---

# 📈 Future Improvements

* Migrate the Data Warehouse to **Amazon Redshift Serverless**
* Add a **real-time streaming pipeline** using Amazon Kinesis + Lambda
* Enhance ETL to support:

  * Sessionization
  * Attribution models
  * User segmentation
* Implement data quality checks & anomaly detection for events
* Introduce a dedicated backend API service for OLTP to remove direct DB exposure

---

# 🧑‍💻 Authors

* **Quốc Hào Triệu** — Project Owner & Architect
* Collaborators: ETL / Data Engineering / Analytics contributors



