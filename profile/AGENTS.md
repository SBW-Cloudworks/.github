# SBW CloudWorks - Agent Context

## 🎯 Project Overview
Clickstream Analytics Platform for E-Commerce - Batch-based ETL system using AWS Serverless architecture with PostgreSQL Data Warehouse and R Shiny Analytics.

---

## ☁️ AWS Services Used

### Compute & Hosting
- **AWS Amplify Hosting** - Next.js frontend hosting (SSR + static assets)
- **AWS Lambda** - Serverless compute for data pipeline (Ingest & ETL functions)
- **Amazon EC2** - Two instances:
  - EC2 #1: OLTP PostgreSQL (Public Subnet)
  - EC2 #2: Data Warehouse PostgreSQL + R Shiny Server (Private Subnet)

### Networking & Security
- **Amazon VPC** - Network isolation with public and private subnets
  - VPC CIDR: `10.0.0.0/16`
  - Public Subnet: `10.0.1.0/24` (OLTP Layer)
  - Private Subnet 1: `10.0.2.0/24` (Analytics Layer)
  - Private Subnet 2: `10.0.3.0/24` (ETL Layer)
- **Internet Gateway** - Public internet connectivity for public subnet
- **VPC Endpoints**:
  - S3 Gateway VPC Endpoint (Private S3 access)
  - SSM Interface Endpoints (Session Manager without SSH)
- **Security Groups** - Firewall rules for EC2 and Lambda

### Storage & Content Delivery
- **Amazon S3** - Two buckets:
  - Static assets bucket (managed by Amplify)
  - Raw clickstream data bucket (partitioned by date/hour)
- **Amazon CloudFront** - CDN for global content distribution

### Data Pipeline & Integration
- **Amazon API Gateway (HTTP API)** - Ingestion endpoint for clickstream events
- **Amazon EventBridge** - Scheduled ETL triggers (cron-based batch processing)

### Identity & Access Management
- **Amazon Cognito** - User authentication and identity management
- **AWS IAM** - Access control and roles for Lambda functions

### Monitoring & Operations
- **Amazon CloudWatch** - Logging and monitoring for:
  - API Gateway access logs
  - Lambda execution logs
  - VPC Flow Logs (optional)
- **AWS Systems Manager (Session Manager)** - Zero-SSH admin access via VPC interface endpoints

---

## 🏗️ Architecture Patterns

### Three-Layer Design
1. **User-Facing Domain**
   - Frontend: Next.js on Amplify
   - OLTP DB: PostgreSQL on EC2 (Public Subnet)
   - Auth: Cognito

2. **Ingestion & Data Lake Domain**
   - API Gateway → Lambda Ingest → S3 Raw Bucket
   - EventBridge → Lambda ETL (VPC-enabled)

3. **Analytics Domain**
   - PostgreSQL Data Warehouse (Private EC2)
   - R Shiny Server (Private EC2)
   - Admin access via Session Manager

### Key Technical Decisions
- **No NAT Gateway** - Cost optimization via S3 Gateway VPC Endpoint
- **Batch ETL** - EventBridge scheduled processing (e.g., every 30 minutes)
- **Private Analytics** - DW and Shiny in private subnet, no public IP
- **Direct OLTP Access** - Amplify connects to OLTP EC2 via Internet Gateway using Prisma
- **Session Manager Only** - No SSH/bastion, admin tunneling via SSM interface endpoints

---

## 📊 Data Flow

1. User → CloudFront → Amplify → Cognito (Auth)
2. Frontend → API Gateway → Lambda Ingest → S3 Raw Bucket
3. EventBridge (cron) → Lambda ETL → S3 (via VPC Endpoint) → DW (via VPC internal)
4. R Shiny → DW (localhost) → Analytics Dashboards
5. Admin → Session Manager (VPC Endpoint) → DW/Shiny EC2 (port forwarding)

---

## 💾 Databases

- **PostgreSQL OLTP** (EC2, Public) - E-commerce operational data
- **PostgreSQL Data Warehouse** (EC2, Private) - Clickstream analytics data

---

## 🔑 Cost Optimization Features

- S3 for raw data storage (instead of database)
- Lambda serverless compute (pay per execution)
- No NAT Gateway (S3 Gateway VPC Endpoint)
- EventBridge scheduled triggers (batch vs real-time)
- Single EC2 for DW + Shiny (consolidated analytics)
