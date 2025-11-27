![sbwCloudworks](swbCloudworksBanner.png)

## Technical Architecture Diagram
![ClickStreamDiagramV8](ClickStreamDiagramV8.png)
# Clickstream Analytics Platform – Batch Processing Architecture

## 🌐 Overview

This project implements a **Clickstream Analytics System** using AWS services with a **Batch Processing Architecture**. It handles data collection, raw storage, periodic ETL processing, and analytics visualization using a self-managed PostgreSQL + R Shiny Server running on EC2.

The system emphasizes **low cost**, **scalability**, **security**, and **full control of the data warehouse layer**.

---

## 📌 1. Architecture Components

The system is built using the following AWS services:

* **Frontend Hosting:** AWS Amplify Hosting (CloudFront integrated)
* **Authentication:** Amazon Cognito (User Pool)
* **API Layer:** Amazon API Gateway (HTTP API)
* **Data Ingestion:** AWS Lambda (Clickstream ingest)
* **Raw Data Lake:** Amazon S3 (Raw Layer)
* **Batch Scheduler:** Amazon EventBridge (Cron Job)
* **ETL Processor:** AWS Lambda ETL
* **Private Connectivity:** VPC Endpoint Interface
* **Internal Routing:** Internal ALB
* **Data Warehouse & Analytics:** EC2 running PostgreSQL + R Shiny Server
* **Visualization:** Shiny Dashboard

---

## 🔄 2. Detailed Data Flow

Below is the complete data flow of the system:

### **(1) User → Amplify Hosting**

Users access the website hosted on **Amplify Hosting**. Amplify includes CloudFront + S3 internally.

### **(2) Amplify → Cognito Authentication**

Frontend calls Cognito for:

* Login / Registration
* Receiving JWT tokens (ID, Access, Refresh)

Tokens are stored on the client for authenticated requests.

### **(3) Frontend → API Gateway**

Frontend calls the API Gateway endpoint, sending JWT tokens. API Gateway verifies tokens using Cognito Authorizer.

### **(4) API Gateway → Lambda Ingest**

Lambda Ingest receives clickstream events:

* Normalizes JSON
* Adds metadata
* Generates session identifiers
* Prepares data for Raw Layer

### **(5) Lambda Ingest → S3 Raw Layer**

Lambda stores data into partitioned S3 structure:

```
s3://clickstream/raw/YYYY/MM/DD/HH/*.json
```

This forms the Raw Data Lake.

### **(6) EventBridge → Trigger Lambda ETL**

EventBridge triggers the ETL Lambda on a schedule (e.g., hourly batch).

### **(7) Lambda ETL → Read Raw Data**

ETL Lambda:

* Reads JSON from S3 Raw Layer
* Validates and aggregates events
* Converts JSON → SQL rows
* Normalizes schema fields

### **(8) Lambda ETL → VPC Endpoint Interface**

Lambda does **not** run inside VPC, but needs access to EC2 → therefore uses **VPC Endpoint Interface**.

Traffic flows:

```
Lambda → VPC Endpoint → Internal ALB → EC2
```

### **(9) Internal ALB → Forward to EC2**

Internal ALB ensures private-only routing and forwards processed data to EC2.

### **(10) EC2 (PostgreSQL + Shiny)**

EC2 serves dual roles:

* **PostgreSQL Database (self-managed)**
* **R Shiny Server for analytics visualization**

Data is inserted into PostgreSQL for querying and dashboard rendering.

### **(11) Admin → View Shiny Dashboard**

Admins access the dashboard hosted on EC2 to view processed analytics such as:

* User behavior patterns
* Page performance metrics
* Conversion funnels
* Traffic sources
* Session duration
* Retention insights

---

## 🗂 3. Architecture Summary Diagram

High-level system flow:

```
User
→ Amplify Hosting
→ Cognito
→ API Gateway
→ Lambda Ingest
→ S3 Raw Layer
→ EventBridge Cron
→ Lambda ETL
→ VPC Endpoint Interface
→ Internal ALB
→ EC2 (PostgreSQL + Shiny)
→ Admin Dashboard
```

---

## 🏗 4. Design Justification

### ✔ Amplify Hosting

* Automatic CI/CD
* CloudFront + S3 integrated
* No server maintenance

### ✔ Cognito Authentication

* Secure JWT workflow
* Easy integration with API Gateway

### ✔ Serverless Ingestion (API Gateway + Lambda)

* Low cost
* Automatically scalable

### ✔ S3 Raw Layer

* Durable, cheap, ideal for Data Lake

### ✔ EventBridge Batch Scheduling

* Flexible cron
* Ideal for periodic ETL processing

### ✔ Lambda ETL

* Stateless, scalable ETL jobs
* Converts NoSQL → SQL

### ✔ VPC Endpoint + Internal ALB

* Ensures secure private network communication
* No exposure of EC2 to the internet

### ✔ EC2 PostgreSQL + Shiny

* Full control of Data Warehouse
* Ideal for data analytics dashboards

---

## 💾 5. Project Folder Structure (Recommended)

```
📦 Clickstream-Analytics
 ┣ 📂 infrastructure
 ┃ ┗ 📜 terraform
 ┣ 📂 frontend
 ┃ ┗ 📜 React/NextJS source
 ┣ 📂 lambda
 ┃ ┣ 📜 ingest.py
 ┃ ┗ 📜 etl.py
 ┣ 📂 scripts
 ┃ ┗ 📜 ec2-setup.sh
 ┣ 📂 shiny
 ┃ ┗ 📜 app.R
 ┗ 📜 README.md
```

---

## 🚀 6. Deployment Workflow

1. Deploy Amplify Hosting
2. Configure Cognito User Pool
3. Create API Gateway HTTP API
4. Deploy Lambda Ingest and ETL
5. Create S3 Raw Layer bucket
6. Set up EventBridge cron
7. Create VPC Endpoint + Internal ALB
8. Launch EC2 and install PostgreSQL + Shiny
9. Configure ALB → EC2 routing
10. Test ingestion → ETL → database workflow
11. Access the Shiny dashboard

---

## ⭐ Author

Clickstream Analytics System
Developed by **Trieu Quoc Hao (SBW Team)**
