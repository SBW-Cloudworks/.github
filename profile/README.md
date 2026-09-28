![SBW Cloudworks](swbCloudworksBanner.png)

<h1 align="center">Clickstream Analytics Platform for E-Commerce</h1>

<p align="center"><b>Turn raw shopper behavior into trusted, hourly business insight - on a private, cost-first AWS pipeline.</b></p>

<p align="center">Batch ETL • AWS Serverless • PostgreSQL Data Warehouse • R Shiny Analytics</p>

<p align="center"><a href="https://github.com/SBW-Cloudworks/ClickSteam.NextJS">Storefront</a> • <a href="https://github.com/SBW-Cloudworks/ClickStream.Lambda">Data pipeline</a> • <a href="https://github.com/SBW-Cloudworks/ClickSteam.RShiny">Dashboard</a> • <a href="https://github.com/SBW-Cloudworks/ClickStream.LocalStack-Terraform">Local environment</a> • <a href="https://the-khiem7.github.io/AWS.FirstCloudJourney/5-workshop/">Workshop</a></p>

![Clickstream architecture diagram](ClickStreamDiagramV11.png)

**On this page:** [The problem](#-the-problem) · [How we solve it](#-how-we-solve-it) · [What it costs](#-what-it-costs) · [How the pipeline runs](#-how-the-pipeline-runs) · [AWS services and the problem each solves](#-aws-services-and-the-problem-each-solves) · [Technical deep dive](#-technical-deep-dive) · [Roadmap](#-roadmap) · [Repositories](#-repositories)

---

## The problem

**Every click tells a story. Most online shops cannot read it.**

E-commerce teams rarely lack data. What they lack is a reliable way to turn scattered shopper behavior into decisions - at the right time, in the right context, and on infrastructure they can actually run. Once a business takes behavior-driven marketing seriously, the same questions keep coming up:

**"Where exactly do shoppers give up?"**

Product views, clicks, add-to-cart and checkout are recorded in different places and in different shapes. Nobody can draw the full journey, so the drop-off points stay a guess.

**"Why did last week's numbers change?"**

Every feature and every release sends events its own way. The same report gives different answers over time, and the business slowly stops trusting the dashboard.

**"Can we fix this report without breaking history?"**

With no original copy of the data underneath, every change to the processing logic is a gamble. Numbers cannot be audited, errors cannot be traced, and past data cannot be replayed.

**"Can analytics run safely inside a locked-down network?"**

Real environments keep databases in private networks, with no SSH and strict isolation. Setups that work in a demo often break once the doors are closed.

**"Do we really need a streaming platform for hourly decisions?"**

Always-on streaming stacks, NAT gateways and managed warehouses cost far more than a team that decides hour by hour can justify.

> **Scope.** The storefront - sign-in, browsing, cart and checkout - is the foundation of a working e-commerce site. The heart of this project is what we built on top of it: the **clickstream pipeline**, **batch processing**, **data warehouse** and **analytics dashboard**.

<p align="center"><img src="assets/storefront.png" alt="SHOPCART storefront home page" width="85%"><br><sub>The SHOPCART storefront - every browse, click and add-to-cart here becomes a clickstream event.</sub></p>

---

## How we solve it

We built a **batch-based clickstream analytics platform** that puts data discipline, day-to-day operability and cost ahead of novelty. Each answer below mirrors one of the questions above.

**One event language, from browser to dashboard.**

A single publisher built into the storefront sends every interaction in the same shape, tied to the same shopper, browser and session. For the first time, the shopper journey can be drawn end to end.

**The same numbers, every release.**

A fixed event contract is enforced twice - once in the browser and once in the warehouse schema - and every event is stamped with one trusted server clock. Last week's report means the same thing today.

**Every raw click is kept.**

Events land untouched in Amazon S3, filed by hour, before anything transforms them. Any report can be rebuilt and any hour replayed, without ever touching the original data.

<table>
<tr>
<td width="50%" align="center"><img src="assets/raw-event.png" alt="Raw clickstream event as stored in Amazon S3"></td>
<td width="50%" align="center"><img src="assets/warehouse-row.png" alt="The same kind of event as a curated row in the data warehouse"></td>
</tr>
<tr>
<td align="center"><sub>Raw event in S3 - the original payload plus server metadata</sub></td>
<td align="center"><sub>Curated row in the warehouse - one fixed schema for every event (IDs blurred)</sub></td>
</tr>
</table>

**Private by default, and still easy to run.**

The warehouse, the processing and the dashboard have no public address. The team manages them through AWS Systems Manager without SSH, and an email alert goes out as soon as a pipeline run fails.

**Hourly batches, pay-per-use.**

Processing runs once an hour - the pace at which marketing and product teams actually make decisions - on serverless compute that costs nothing while idle. No NAT gateway, no streaming cluster.

### What the business sees

The R Shiny dashboard reads straight from the data warehouse and refreshes every 10 seconds:

- **KPI cards** - Total events, Unique clients, Checkout rate (`checkout_complete` ÷ `add_to_cart_click`).
- **Overview** - Events over time, Top 10 event types, Events by login state (logged-in vs anonymous shoppers).
- **Products** - Top categories and Top brands by engagement.
- **Raw sample** - the latest product events, paged and newest first, for spot checks.
- **Filters** - date range and user login state.

<p align="center"><img src="assets/dashboard-overview.png" alt="SBW Clickstream Dashboard - Overview tab"><br><sub>Overview tab - KPI cards, events over time, top event types and events by login state</sub></p>

<p align="center"><img src="assets/dashboard-products.png" alt="SBW Clickstream Dashboard - Products tab"><br><sub>Products tab - top categories and top brands by shopper engagement</sub></p>

---

## What it costs

Cost-first is not a slogan - these are the real AWS bills from our development and demo period in December 2025.

| Item                                                    | Cost                |
| ------------------------------------------------------- | ------------------- |
| **Total development cost**                        | **36.29 USD** |
| AWS Amplify Hosting (storefront)                        | 5.00 USD / month    |
| Amazon EC2 t3.small, 15 GB (data warehouse + dashboard) | 10.20 USD / month   |
| Amazon EC2 t3.small, 8 GB (store database)              | 9.36 USD / month    |

The serverless pieces - API Gateway, both Lambda functions, EventBridge and S3 - barely register on the bill, because they cost nothing while the store is quiet.

<p align="center"><img src="assets/cost-daily.png" alt="Daily AWS costs by service, 2 to 7 December 2025"><br><sub>Daily AWS cost by service during development (2-7 December 2025)</sub></p>

---

## How the pipeline runs

![Clickstream architecture diagram V11](ClickStreamDiagramV11.png)

The numbers below match the numbered circles in the diagram.

### A. Shop - the storefront foundation

| Step | What happens                                                                                                           | Services                                               |
| ---- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| 1    | The shopper signs in. Cognito issues tokens, and the shopper's Cognito user ID becomes`userId` on every later event. | Amazon Cognito                                         |
| 2    | The shopper opens the store. CloudFront serves the Next.js app hosted on Amplify.                                      | Amazon CloudFront, AWS Amplify Hosting                 |
| 3    | Product images and media load from S3.                                                                                 | Amazon S3 (media assets)                               |
| 4    | Server-side pages read products, carts and orders from the OLTP PostgreSQL database through the Internet Gateway.      | AWS Amplify (SSR), Internet Gateway, Amazon EC2 (OLTP) |

### B. Collect - every interaction becomes a raw event

| Step | What happens                                                                                                                                                                                          | Services                                    |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| 5    | The ClickStream publisher in the browser sends each interaction as one JSON event, with client, session, user and product context attached.                                                           | ClickStream publisher → Amazon API Gateway |
| 7    | API Gateway receives`POST /clickstream` and invokes Lambda Ingest.                                                                                                                                  | Amazon API Gateway (HTTP API), AWS Lambda   |
| 8    | Lambda Ingest checks the request, adds server metadata (receive time, request ID, trace ID, source IP, user agent) and writes the event unchanged to`events/YYYY/MM/DD/HH/event-<uuid>.json` (UTC). | AWS Lambda (Ingest), Amazon S3 (raw)        |

### C. Transform - hourly batch ETL into the warehouse

| Step | What happens                                                                                                                                                       | Services                                                  |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| 9    | Each closed UTC hour of raw events becomes one batch for the scheduler.                                                                                            | Amazon S3 (raw)                                           |
| 10   | At minute 5 of every hour, EventBridge triggers Lambda ETL for the previous full hour.                                                                             | Amazon EventBridge                                        |
| 11   | Lambda ETL runs in the private subnet and reads that hour's raw objects through the S3 Gateway Endpoint - no internet path.                                        | S3 Gateway VPC Endpoint, AWS Lambda (ETL)                 |
| 12   | ETL maps each event to the warehouse contract, flattens the product context and inserts rows into`clickstream_events`, skipping any `event_id` already loaded. | AWS Lambda (ETL), PostgreSQL Data Warehouse on Amazon EC2 |

### D. Analyze and operate

| Step | What happens                                                                                                                                      | Services                                  |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| 15   | R Shiny, on the same instance as the warehouse, queries PostgreSQL over localhost and renders the dashboard.                                      | R Shiny Server, PostgreSQL Data Warehouse |
| 13   | An admin opens a Session Manager session - no SSH keys, no bastion host.                                                                          | AWS Systems Manager Session Manager       |
| 14   | Session traffic reaches the private EC2 instance through the SSM interface endpoint, with port forwarding to the dashboard or the database.       | SSM Interface VPC Endpoint, Amazon EC2    |
| 6    | API Gateway and both Lambda functions log to CloudWatch. Alarms email the IT/ops team through SNS. IAM roles limit what each component can touch. | Amazon CloudWatch, Amazon SNS, AWS IAM    |

### Life of one click

```mermaid
sequenceDiagram
    actor Shopper
    participant Pub as ClickStream publisher
    participant API as API Gateway
    participant Ing as Lambda Ingest
    participant Raw as S3 raw bucket
    participant EB as EventBridge
    participant ETL as Lambda ETL
    participant DW as PostgreSQL DW
    participant Dash as R Shiny
    actor Analyst
    Shopper->>Pub: Clicks Add to cart at 14:20 UTC
    Pub->>API: POST /clickstream (add_to_cart_click)
    API->>Ing: Invoke
    Ing->>Raw: Write events/2026/09/28/14/event-uuid.json
    Note over Raw: Hour 14 keeps filling until 15:00 UTC
    EB->>ETL: 15:05 UTC - process hour 14
    ETL->>Raw: List and read hour 14 via S3 Gateway Endpoint
    ETL->>DW: Insert rows, skip duplicate event_id
    Dash->>DW: Query every 10 seconds
    Dash->>Analyst: Updated KPIs and charts
```

---

## 🧩 AWS services and the problem each solves

Every service in the platform earns its place by removing a concrete problem for the business.

### Shop - the storefront foundation

| Service                                | What it does for us                                                       | The problem it removes                                                                            |
| -------------------------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **Amazon Cognito**               | Signs shoppers up and in.                                                 | Behavior is tied to a real customer across sessions and devices, not to an anonymous browser tab. |
| **AWS Amplify Hosting**          | Builds and hosts the storefront, with the ClickStream publisher built in. | Every page reports behavior the same way, and there are no web servers to run.                    |
| **Amazon CloudFront**            | Delivers the storefront to shoppers.                                      | Pages load fast wherever the shopper is.                                                          |
| **Amazon S3 (media assets)**     | Stores product images and media.                                          | Store content is kept safely at low cost.                                                         |
| **Amazon EC2 (OLTP PostgreSQL)** | Runs the store's day-to-day database for users, products and orders.      | Orders and checkout stay apart from analytics, so reports never slow down sales.                  |

### Collect - every interaction becomes a raw event

| Service                                 | What it does for us                                      | The problem it removes                                                               |
| --------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **Amazon API Gateway (HTTP API)** | Gives every event one public door,`POST /clickstream`. | All behavior data enters in one place, and cost grows only with traffic.             |
| **AWS Lambda (Ingest)**           | Checks each event, stamps the server time and stores it. | Every event runs on one trusted clock, and nothing is paid while the store is quiet. |
| **Amazon S3 (raw clickstream)**   | Keeps every raw event, filed by hour.                    | The original data is always there to audit, debug or rebuild any report.             |

### Transform - hourly processing into the warehouse

| Service                           | What it does for us                                     | The problem it removes                                                                                  |
| --------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **Amazon EventBridge**      | Starts processing at minute 5 of every hour.            | No scheduler server to keep alive, and every run covers a fixed hour that can be replayed.              |
| **AWS Lambda (ETL)**        | Turns one hour of raw events into clean warehouse rows. | Every event follows the same schema, duplicates are skipped, and any hour can be reprocessed on demand. |
| **S3 Gateway VPC Endpoint** | Gives processing a private road to the raw data.        | Data never crosses the internet, and no NAT gateway is needed.                                          |

### Analyze and operate

| Service                                        | What it does for us                                                   | The problem it removes                                                                   |
| ---------------------------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Amazon VPC (private subnet)**          | Hosts the warehouse, dashboard and processing with no public address. | Analytics data cannot be reached from the internet.                                      |
| **Amazon EC2 (PostgreSQL DW + R Shiny)** | Runs the data warehouse and the dashboard on one server.              | The whole customer journey is answered in one place, on a single low-cost server.        |
| **AWS Systems Manager Session Manager**  | Lets admins work on the private server through private endpoints.     | Safe maintenance with no SSH keys, no bastion host and no open ports.                    |
| **AWS IAM**                              | Gives each function only the access it needs.                         | No pipeline step can alter the original raw data.                                        |
| **Amazon CloudWatch**                    | Collects logs and metrics, and raises alarms.                         | The team sees which step failed and why, instead of digging through servers.             |
| **Amazon SNS**                           | Emails the IT/ops team when an alarm fires.                           | Nobody has to watch logs, and a failed hour is fixed and replayed before data is missed. |

---

## 🔬 Technical deep dive

Each section below is collapsed. Expand the ones you need.

<details>
<summary><b>Event contract - what every event carries</b></summary>

<br>

**Events the storefront emits**

| Event                      | Triggered when                                                           |
| -------------------------- | ------------------------------------------------------------------------ |
| `page_view`              | Any route or query-string change                                         |
| `click`                  | Any click on the page, with the clicked element's tag, id, role and text |
| `home_view`              | The home page is shown                                                   |
| `category_view`          | A category page is shown                                                 |
| `product_view`           | A product detail page is shown                                           |
| `add_to_cart_click`      | The shopper clicks Add to cart                                           |
| `remove_from_cart_click` | The shopper removes an item from the cart                                |
| `wishlist_toggle`        | The shopper adds or removes a wishlist item                              |
| `checkout_start`         | The shopper starts checkout (cart total and item count attached)         |
| `checkout_complete`      | Checkout finishes                                                        |

**Fields sent by the browser publisher**

| Field                     | Meaning                                                                                         |
| ------------------------- | ----------------------------------------------------------------------------------------------- |
| `eventId`               | UUID generated per event, used by the ETL to skip duplicates                                    |
| `eventName`             | One of the events above                                                                         |
| `userId`                | Cognito user ID when signed in, otherwise null                                                  |
| `userLoginState`        | `logged_in` or `anonymous`                                                                  |
| `clientId`              | Stable browser ID, kept in`localStorage`                                                      |
| `sessionId`             | Session ID, kept in`sessionStorage`, renewed after 30 minutes of inactivity                   |
| `isFirstVisit`          | `true` on the first event of a new browser                                                    |
| `pageUrl`, `referrer` | Page context                                                                                    |
| `product`               | `id`, `name`, `category`, `brand`, `price`, `discountPrice`, `urlPath`            |
| `element`               | Clicked element details, with text from password, email, phone and number inputs never captured |

**Warehouse table `clickstream_events`**

| Column                                                                                                    | Type              |
| --------------------------------------------------------------------------------------------------------- | ----------------- |
| `event_id`                                                                                              | UUID, primary key |
| `event_timestamp`                                                                                       | TIMESTAMPTZ       |
| `event_name`                                                                                            | TEXT              |
| `user_id`, `user_login_state`, `identity_source`, `client_id`, `session_id`                     | TEXT              |
| `is_first_visit`                                                                                        | BOOLEAN           |
| `context_product_id`, `context_product_name`, `context_product_category`, `context_product_brand` | TEXT              |
| `context_product_price`, `context_product_discount_price`                                             | BIGINT            |
| `context_product_url_path`                                                                              | TEXT              |

Full details: [ClickSteam.NextJS](https://github.com/SBW-Cloudworks/ClickSteam.NextJS)

</details>

<details>
<summary><b>Ingestion internals - API Gateway and Lambda Ingest</b></summary>

<br>

- **Endpoint:** API Gateway HTTP API, route `POST /clickstream`, Lambda proxy integration.
- **Checks:** the route must be `POST /clickstream`, a body must be present, and the body must be valid JSON. Invalid requests get `400`; unexpected errors get `500`.
- **Server metadata:** the original payload is kept as-is, and an `_ingest` object is added with `receivedAt` (server time, ISO UTC), `sourceIp`, `userAgent`, `method`, `path`, `requestId`, `apiId`, `stage` and `traceId`.
- **Storage layout:** one S3 object per event, at `s3://<raw-clickstream-bucket>/events/YYYY/MM/DD/HH/event-<uuid>.json`, with all partitions in UTC.
- **Network:** Lambda Ingest runs outside the VPC, so it writes to S3 without VPC endpoints or a NAT gateway.
- **Runtime:** Node.js with AWS SDK for JavaScript v3.

Full details: [ClickStream.Lambda](https://github.com/SBW-Cloudworks/ClickStream.Lambda)

</details>

<details>
<summary><b>Batch ETL internals - EventBridge and Lambda ETL</b></summary>

<br>

- **Schedule:** EventBridge rule `cron(5 * * * ? *)` - minute 5 of every hour.
- **Window:** by default the previous full UTC hour (`events/YYYY/MM/DD/HH/`).
- **Manual re-runs:** invoke the function with `{"hoursBack": N}`, `{"targetUtcHour": "2025-12-06T05:00:00Z"}` or `{"overridePrefix": "events/2025/12/06/05/"}` to reprocess any hour or day from raw.
- **Mapping:** accepts camelCase or snake_case field names, flattens `product` into the `context_product_*` columns and converts prices to numbers.
- **Timestamp:** server receive time (`_ingest.receivedAt`) first, then the client timestamp, then the S3 object time - one trusted clock for every event.
- **Load:** multi-row `INSERT ... ON CONFLICT (event_id) DO NOTHING`, one transaction per batch, with a configurable batch size (default 200). The table is created on first run if it does not exist.
- **Network:** VPC-enabled Lambda in the private subnet. It reads S3 through the S3 Gateway Endpoint and reaches PostgreSQL on port 5432 over the VPC's internal routing.
- **Runtime:** Node.js with AWS SDK for JavaScript v3 and `pg`.

Full details: [ClickStream.Lambda](https://github.com/SBW-Cloudworks/ClickStream.Lambda)

</details>

<details>
<summary><b>Network and security design</b></summary>

<br>

**VPC layout** - VPC CIDR `10.0.0.0/16`

| Subnet                                     | Contents                                                                                                                                              | Internet access                                                                           |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **Public subnet - OLTP layer**       | EC2 PostgreSQL OLTP (public IP)                                                                                                                       | Default route`0.0.0.0/0` to the Internet Gateway, so Amplify SSR can reach the database |
| **Private subnet - Analytics layer** | EC2 PostgreSQL DW + R Shiny Server (no public IP), Lambda ETL network interfaces, SSM interface endpoints (`ssm`, `ssmmessages`, `ec2messages`) | None - no route to the Internet Gateway                                                   |

**Routing**

| Route table         | Routes                                                              |
| ------------------- | ------------------------------------------------------------------- |
| Public              | `10.0.0.0/16` → local, `0.0.0.0/0` → Internet Gateway         |
| Private (Analytics) | `10.0.0.0/16` → local, S3 prefix list → S3 Gateway VPC Endpoint |

**No NAT gateway.** Private components reach S3 only through the S3 Gateway VPC Endpoint and reach Systems Manager only through the SSM interface endpoints.

**Security groups**

| Security group                        | Inbound                                                                                 | Outbound                                   |
| ------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------ |
| **SG-OLTP**                     | `5432/tcp` from Amplify / trusted ranges (Prisma), `22/tcp` from the admin IP       | Default                                    |
| **SG-DW** (DW + Shiny instance) | `5432/tcp` from SG-ETL-Lambda. No inbound SSH. Shiny reads PostgreSQL over localhost. | `443/tcp` to the SSM interface endpoints |
| **SG-ETL-Lambda**               | None                                                                                    | S3 Gateway Endpoint, SG-DW on`5432/tcp`  |

**Admin access** - the DW + Shiny instance runs the SSM Agent. Admins open Session Manager port-forwarding sessions to the dashboard (port `3838`) or PostgreSQL (port `5432`) and use them from `localhost` on their own machine.

**Managed services outside the VPC**

| Service                            | How it connects                                                               |
| ---------------------------------- | ----------------------------------------------------------------------------- |
| AWS Amplify Hosting                | Reaches the OLTP instance through the Internet Gateway using Prisma           |
| Amazon CloudFront                  | Serves the storefront. No VPC interaction.                                    |
| Amazon Cognito                     | Called by the storefront over HTTPS. No VPC interaction.                      |
| Amazon API Gateway + Lambda Ingest | Write raw events straight to S3. No VPC configuration.                        |
| Amazon EventBridge                 | Invokes the VPC-enabled Lambda ETL                                            |
| AWS Systems Manager                | Regional control plane, reached privately through the SSM interface endpoints |

</details>

<details>
<summary><b>IAM and observability</b></summary>

<br>

**IAM - one role per function**

| Role           | Permissions                                                                                                       |
| -------------- | ----------------------------------------------------------------------------------------------------------------- |
| Lambda Ingest  | `s3:PutObject` on `<raw-bucket>/events/*`, CloudWatch Logs write                                              |
| Lambda ETL     | `s3:GetObject` and `s3:ListBucket` on the raw bucket, VPC network-interface management, CloudWatch Logs write |
| DW + Shiny EC2 | Instance profile that registers the instance with Systems Manager                                                 |

**Monitoring and alerting**

- CloudWatch Logs for API Gateway access logs and for both Lambda functions. Each ETL run logs how many events it processed and inserted.
- CloudWatch alarms on the pipeline publish to an SNS topic that emails the IT/ops team, so a failed stage is noticed without anyone watching the logs.
- Session Manager sessions can be audited through CloudWatch or S3.
- VPC Flow Logs are optional, for network traffic analysis.

</details>

<details>
<summary><b>Storage - two S3 buckets</b></summary>

<br>

| Bucket                    | Contents                                                                 |
| ------------------------- | ------------------------------------------------------------------------ |
| **Media assets**    | Product images and static media served to the storefront                 |
| **Raw clickstream** | Raw JSON events from Lambda Ingest, partitioned`events/YYYY/MM/DD/HH/` |

There is no separate "processed" bucket. Curated data lives in the PostgreSQL data warehouse, and the raw bucket stays the source it can be rebuilt from.

</details>

<details>
<summary><b>Tech stack</b></summary>

<br>

| Layer             | Technology                                                                                                                             |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Storefront        | Next.js 15, React 18, TypeScript, Tailwind CSS, Prisma, Amplify Auth with Amazon Cognito                                               |
| Data pipeline     | Node.js AWS Lambda functions, AWS SDK for JavaScript v3,`pg`                                                                         |
| Databases         | PostgreSQL on EC2 for OLTP, PostgreSQL on EC2 for the data warehouse                                                                   |
| Analytics         | R Shiny Server with`shiny`, `DBI`, `RPostgres`, `pool`, `dplyr`, `ggplot2`, `lubridate`                                  |
| AWS               | Amplify Hosting, CloudFront, Cognito, S3, API Gateway (HTTP API), Lambda, EventBridge, EC2, VPC, IAM, CloudWatch, SNS, Systems Manager |
| Local environment | LocalStack, Terraform                                                                                                                  |

</details>

<details>
<summary><b>Local development with LocalStack</b></summary>

<br>

The [ClickStream.LocalStack-Terraform](https://github.com/SBW-Cloudworks/ClickStream.LocalStack-Terraform) repository emulates the core pipeline on LocalStack with Terraform: S3, Lambda, API Gateway, EventBridge, Cognito and the VPC.

LocalStack may create extra internal buckets, but the platform itself depends on only the two buckets above. Amplify Hosting, Cognito hosted UI flows and full VPC networking are only partly supported by LocalStack and are tested in real AWS.

</details>

---

## 🧭 Roadmap

The platform is batch-first today. The next steps grow what the analytics layer can tell the business:

- **AI-assisted analytics in R Shiny** - trend and anomaly detection, behavior-based customer segmentation, and automatic insights such as "which product group is rising" or "which step is dropping unusually".
- **Session funnels and journey views** - conversion funnels and path analysis built on the session and client IDs every event already carries.
- **Data quality checks** - validation and anomaly checks on incoming events before they reach the warehouse.
- **Real-time stream** - Amazon Kinesis alongside the batch path for use cases that need second-level freshness.
- **Amazon Redshift Serverless** - a managed warehouse option as data volume grows.

---

## 📦 Repositories

| Repository                                                                                            | What it holds                                                     |
| ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| [ClickSteam.NextJS](https://github.com/SBW-Cloudworks/ClickSteam.NextJS)                               | Next.js storefront, Cognito sign-in and the ClickStream publisher |
| [ClickStream.Lambda](https://github.com/SBW-Cloudworks/ClickStream.Lambda)                             | Lambda Ingest and Lambda ETL source code and deployment guide     |
| [ClickSteam.RShiny](https://github.com/SBW-Cloudworks/ClickSteam.RShiny)                               | R Shiny dashboard and its setup on a private EC2 instance         |
| [ClickStream.LocalStack-Terraform](https://github.com/SBW-Cloudworks/ClickStream.LocalStack-Terraform) | LocalStack and Terraform environment for local development        |

📘 **Step-by-step workshop:** [AWS First Cloud Journey - Clickstream workshop](https://the-khiem7.github.io/AWS.FirstCloudJourney/5-workshop/)

---

## 👥 The team

![SBW Cloudworks members](SBWMember.png)
