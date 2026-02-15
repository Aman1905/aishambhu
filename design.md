# Agri-Chakra: MVP System Design

## Architecture Overview

**Design Philosophy**: Fully serverless, event-driven AWS architecture for rapid MVP deployment with minimal operational overhead and cost optimization.

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    USER LAYER                                │
├───────────────────┬─────────────────────────────────────────┤
│ Web/Mobile App    │  WhatsApp Bot (Optional)                │
│ (AWS Amplify)     │  (Twilio + Lambda)                      │
└────────┬──────────┴──────────┬──────────────────────────────┘
         │                     │
         └─────────┬───────────┘
                   │
         ┌─────────▼─────────┐
         │ Amazon Cognito    │ ← Authentication
         │ (Phone OTP)       │
         └─────────┬─────────┘
                   │
         ┌─────────▼─────────┐
         │ Amazon API        │ ← API Layer
         │ Gateway           │
         └─────────┬─────────┘
                   │
         ┌─────────▼─────────┐
         │ AWS Lambda        │ ← Business Logic
         │ (Node.js/Python)  │
         └───┬───┬───┬───┬───┘
             │   │   │   │
    ┌────────┘   │   │   └────────┐
    │            │   │            │
┌───▼──┐  ┌─────▼──┐ ┌──▼─────┐ ┌▼────────┐
│ S3   │  │Dynamo  │ │Rekog   │ │Bedrock  │
│Image │  │DB      │ │Grading │ │Chatbot  │
└──────┘  └────────┘ └────────┘ └─────────┘
                         │
                    ┌────▼──────┐
                    │SageMaker  │
                    │Prediction │
                    └───────────┘
```

---

## 2. Component Details

### 2.1 Frontend (Web/Mobile App - AWS Amplify)

**Technology**:
- Web: React.js (hosted on Amplify)
- Mobile: React Native (APK/IPA)

**Key Features**:
- Farmer flow: Register → Upload Images → Get Grade → Create Listing
- Buyer flow: Search Listings → View Details → Book/Contact
- Real-time updates via WebSocket (API Gateway)

**Hosting**: AWS Amplify
- Auto-deployment from Git
- Built-in CI/CD
- Global CDN distribution
- SSL certificate included

---

### 2.2 Authentication (Amazon Cognito)

**Configuration**:
```yaml
User Pool: agri-chakra-users
Sign-in Method: Phone number (SMS OTP)
MFA: Disabled (for MVP simplicity)
Password Policy: Not applicable (passwordless)

User Attributes:
  - phone_number (primary)
  - custom:user_type (farmer/buyer)
  - custom:location
  - name
  - email (optional)

Triggers:
  - Pre-signup: Validate phone format
  - Post-confirmation: Create user in DynamoDB
```

**Authentication Flow**:
1. User enters phone number
2. Cognito sends OTP via SMS
3. User verifies OTP
4. Cognito issues JWT tokens (access + refresh)
5. Frontend stores tokens, includes in API requests

---

### 2.3 API Layer (Amazon API Gateway)

**Configuration**:
```yaml
API Type: REST API
Authorization: Cognito User Pool Authorizer
Stage: prod
Rate Limiting: 1000 req/sec (burst: 2000)
Cache: 5-minute TTL for GET endpoints

Endpoints:
  POST   /auth/register
  POST   /auth/verify
  
  POST   /listings
  GET    /listings
  GET    /listings/{id}
  PUT    /listings/{id}
  
  POST   /grading/upload
  GET    /grading/{jobId}
  
  POST   /transactions
  GET    /transactions/{id}
  
  POST   /chat
```

**CORS Configuration**:
- Allow Origins: `*` (MVP; restrict in production)
- Allow Methods: GET, POST, PUT, DELETE, OPTIONS
- Allow Headers: Authorization, Content-Type

---

### 2.4 Business Logic (AWS Lambda)

**Lambda Functions**:

```yaml
1. UserManagement:
   Runtime: Node.js 20.x
   Memory: 512 MB
   Timeout: 10s
   Triggers: API Gateway
   Actions: Register, profile CRUD

2. ListingService:
   Runtime: Node.js 20.x
   Memory: 512 MB
   Timeout: 15s
   Triggers: API Gateway
   Actions: Create, search, update listings
   
3. GradingOrchestrator:
   Runtime: Python 3.12
   Memory: 1024 MB
   Timeout: 30s
   Triggers: API Gateway (image upload)
   Actions:
     - Store image in S3
     - Invoke Rekognition/SageMaker
     - Save results to DynamoDB
     - Return grade to user

4. PricingEngine:
   Runtime: Python 3.12
   Memory: 512 MB
   Timeout: 10s
   Triggers: EventBridge (daily)
   Actions: Fetch market data, update price predictions

5. ChatbotHandler:
   Runtime: Python 3.12
   Memory: 1024 MB
   Timeout: 20s
   Triggers: API Gateway
   Actions: Process user queries, invoke Bedrock

6. TransactionProcessor:
   Runtime: Node.js 20.x
   Memory: 512 MB
   Timeout: 15s
   Triggers: API Gateway
   Actions: Booking, payment integration
```

**Environment Variables**:
- `DYNAMODB_TABLE_USERS`
- `DYNAMODB_TABLE_LISTINGS`
- `S3_BUCKET_IMAGES`
- `REKOGNITION_PROJECT_ARN`
- `SAGEMAKER_ENDPOINT`
- `BEDROCK_MODEL_ID`

---

### 2.5 Database (Amazon DynamoDB)

**Tables**:

#### Users Table
```yaml
Table: agri-chakra-users
Partition Key: user_id (String)
Attributes:
  - phone_number (String, GSI)
  - user_type (String: farmer/buyer)
  - name (String)
  - location (Map: {lat, long})
  - kyc_status (String)
  - created_at (Number: timestamp)

GSI: phone-index (phone_number as PK)
Capacity: On-Demand
```

#### Listings Table
```yaml
Table: agri-chakra-listings
Partition Key: listing_id (String)
Sort Key: created_at (Number)
Attributes:
  - farmer_id (String)
  - residue_type (String)
  - quality_grade (String: A/B/C)
  - quantity_tons (Number)
  - price_per_ton (Number)
  - images (List of S3 URLs)
  - location (Map: {lat, long})
  - status (String: active/booked/sold)

GSI: farmer-index (farmer_id as PK, created_at as SK)
GSI: status-index (status as PK, created_at as SK)
Capacity: On-Demand
TTL: enabled on expiry_date (auto-delete old listings)
```

#### Grading Results Table
```yaml
Table: agri-chakra-gradings
Partition Key: grading_id (String)
Attributes:
  - listing_id (String)
  - images (List of S3 URLs)
  - residue_type_detected (String)
  - quality_grade (String)
  - confidence (Number: 0-100)
  - quantity_estimate (Number)
  - model_version (String)
  - created_at (Number)

Capacity: On-Demand
```

#### Transactions Table
```yaml
Table: agri-chakra-transactions
Partition Key: transaction_id (String)
Attributes:
  - listing_id (String)
  - buyer_id (String)
  - farmer_id (String)
  - quantity_tons (Number)
  - total_amount (Number)
  - status (String: pending/confirmed/completed)
  - payment_status (String)
  - created_at (Number)

GSI: buyer-index (buyer_id as PK)
GSI: farmer-index (farmer_id as PK)
Capacity: On-Demand
```

---

### 2.6 Storage (Amazon S3)

**Buckets**:

```yaml
agri-chakra-images-prod:
  Purpose: Store farmer-uploaded residue images
  Structure:
    /{user_id}/{timestamp}-{uuid}.jpg
  
  Lifecycle:
    - Delete after 90 days (post-transaction)
  
  CORS: Enabled (for direct browser upload)
  Encryption: SSE-S3
  Versioning: Disabled (MVP)
  
  Access:
    - Public read: No
    - Pre-signed URLs for upload/download

agri-chakra-models-prod:
  Purpose: Store ML model artifacts
  Access: Private (Lambda only)
```

---

### 2.7 AI/ML Services

#### Amazon Rekognition (Quality Grading)

**Custom Labels Project**:
```yaml
Project: agri-residue-quality
Dataset: 
  - 5,000+ labeled images
  - Classes: rice_straw_A, rice_straw_B, rice_straw_C,
            bagasse_A, bagasse_B, bagasse_C, etc.

Training:
  - Model: Transfer learning (AWS backbone)
  - Training time: 4-6 hours
  - Accuracy target: >80%

Inference:
  - Endpoint: Managed by AWS
  - Latency: <2 seconds
  - Pricing: $4 per 1000 inferences
```

**Usage Flow**:
1. Lambda uploads image to S3
2. Lambda calls `rekognition.detect_custom_labels()`
3. Rekognition returns: `[{Name: "bagasse_A", Confidence: 92.5}]`
4. Lambda stores result in DynamoDB

---

#### Amazon Bedrock (GenAI Chatbot)

**Configuration**:
```yaml
Model: Claude 3 Haiku (cost-optimized)
Use Cases:
  - Answer farmer queries ("How to prepare residue for sale?")
  - Explain quality grades
  - Provide market price insights
  - Troubleshoot listing issues

Implementation:
  - User sends chat message via API
  - Lambda constructs prompt with context:
      System: "You are Agri-Chakra assistant..."
      User: "What is Grade A bagasse?"
  - Bedrock returns response
  - Lambda sends back to user

Cost: ~$0.00025 per 1000 tokens (very cheap)
```

---

#### Amazon SageMaker (Yield Prediction - Optional)

**Model**: Price Prediction
```yaml
Training Data:
  - Historical prices from APMC mandis
  - Residue type, quality, location, season
  - Market demand indicators

Model: XGBoost regression
Endpoint: ml.t3.medium (on-demand)
Latency: <500ms

Input: {residue_type, quality, location, date}
Output: {min_price, max_price, confidence}

Training Frequency: Weekly (via EventBridge)
```

---

## 3. Data Flow Diagrams

### 3.1 Farmer Listing Creation Flow

```
┌──────────────┐
│ Farmer App   │
└──────┬───────┘
       │ 1. Click "Create Listing"
       ▼
┌──────────────────┐
│ Upload Images    │ (3-5 photos)
└──────┬───────────┘
       │ 2. POST /grading/upload
       ▼
┌──────────────────┐
│ API Gateway      │
└──────┬───────────┘
       │ 3. Authorize (Cognito JWT)
       ▼
┌──────────────────────┐
│ Lambda: Grading      │
│ Orchestrator         │
└──────┬───────────────┘
       │ 4. Upload to S3
       ▼
┌──────────────────────┐      ┌─────────────────┐
│ S3 Bucket            │─────▶│ Rekognition     │
│ (Images)             │ 5.   │ (Custom Labels) │
└──────────────────────┘      └────────┬────────┘
                                       │ 6. Grade Result
                                       ▼
                              ┌─────────────────┐
                              │ Lambda saves to │
                              │ DynamoDB        │
                              └────────┬────────┘
                                       │ 7. Return jobId
                                       ▼
                              ┌─────────────────┐
                              │ Farmer App      │
                              │ Shows: Grade A  │
                              │ Confidence: 89% │
                              └────────┬────────┘
       │ 8. Farmer enters quantity, price
       ▼
┌──────────────────────┐
│ POST /listings       │
│ {                    │
│   residue_type,      │
│   quantity,          │
│   grade,             │
│   price,             │
│   images             │
│ }                    │
└──────┬───────────────┘
       │ 9. Lambda saves listing
       ▼
┌──────────────────────┐
│ DynamoDB Listings    │
│ Status: Active       │
└──────────────────────┘
```

---

### 3.2 Buyer Search & Booking Flow

```
┌──────────────┐
│ Buyer App    │
└──────┬───────┘
       │ 1. Search filters
       │    (residue_type=bagasse, grade=A, location=Punjab)
       ▼
┌──────────────────┐
│ GET /listings    │
│ ?filters=...     │
└──────┬───────────┘
       │ 2. API Gateway → Lambda
       ▼
┌──────────────────────┐
│ Lambda: Query        │
│ DynamoDB (GSI)       │
└──────┬───────────────┘
       │ 3. Return matching listings
       ▼
┌──────────────┐
│ Buyer sees   │
│ 15 results   │
└──────┬───────┘
       │ 4. Click on listing
       ▼
┌──────────────────┐
│ GET /listings/123│
└──────┬───────────┘
       │ 5. Fetch full details
       ▼
┌──────────────────────┐
│ Buyer views:         │
│ - Images             │
│ - Grade: A (92%)     │
│ - Quantity: 25 tons  │
│ - Price: ₹3,500/ton  │
│ - Farmer contact     │
└──────┬───────────────┘
       │ 6. Click "Book"
       ▼
┌──────────────────┐
│ POST /transactions│
│ {listing_id,     │
│  quantity: 20}   │
└──────┬───────────┘
       │ 7. Lambda creates transaction
       ▼
┌──────────────────────┐
│ DynamoDB Transaction │
│ Status: Pending      │
│                      │
│ Update Listing       │
│ Status: Booked       │
└──────┬───────────────┘
       │ 8. Send notifications (SNS)
       ▼
┌─────────────────┬────────────────┐
│ Farmer SMS:     │ Buyer Email:   │
│ "Your listing   │ "Booking       │
│  has been       │  confirmed"    │
│  booked!"       │                │
└─────────────────┴────────────────┘
```

---

## 4. Security Implementation

### 4.1 Authentication & Authorization

```yaml
Authentication:
  - Cognito User Pool (phone OTP)
  - JWT tokens (access: 1 hour, refresh: 30 days)
  - API Gateway validates tokens

Authorization:
  - Lambda checks user_type (farmer/buyer)
  - Farmers can only edit own listings
  - Buyers can view all listings

API Gateway IAM Roles:
  - Lambda execution role
  - S3 read/write access
  - DynamoDB CRUD access
  - Rekognition/Bedrock invoke
```

### 4.2 Data Protection

```yaml
Encryption at Rest:
  - S3: SSE-S3 (AES-256)
  - DynamoDB: AWS-managed encryption
  - Logs: CloudWatch encrypted

Encryption in Transit:
  - HTTPS only (TLS 1.3)
  - API Gateway enforces HTTPS

Sensitive Data:
  - Phone numbers: Hashed in Cognito
  - No credit card storage (payment gateway handles)
```

### 4.3 Input Validation

```yaml
API Gateway:
  - Request validation (JSON schema)
  - Max request size: 10 MB
  - Rate limiting: 100 req/min per user

Lambda:
  - Sanitize inputs (prevent injection)
  - Image validation: 
      - File type: jpg, png only
      - Max size: 5 MB per image
      - Virus scan (optional for MVP)
```

---

## 5. Monitoring & Observability

### 5.1 CloudWatch Dashboards

```yaml
Metrics to Monitor:
  - API Gateway: Request count, latency, errors
  - Lambda: Invocations, duration, errors, throttles
  - DynamoDB: Consumed capacity, throttles
  - Rekognition: Inference count, errors

Alarms:
  - Lambda errors > 5% (5 min)
  - API latency > 3 seconds
  - DynamoDB throttles > 10
  - S3 upload failures

Logs:
  - CloudWatch Logs for all Lambda functions
  - Retention: 7 days (MVP)
  - Log Insights queries for debugging
```

### 5.2 X-Ray Tracing

```yaml
Enable on:
  - API Gateway
  - Lambda functions
  
Trace Details:
  - Request path visualization
  - Latency breakdown per service
  - Error identification
```

---

## 6. Deployment Strategy

### 6.1 Infrastructure as Code (IaC)

**Tool**: AWS SAM (Serverless Application Model)

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  # API Gateway
  AgriChakraAPI:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: CognitoAuthorizer
  
  # Lambda Functions
  ListingFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: nodejs20.x
      Handler: index.handler
      Environment:
        Variables:
          DYNAMODB_TABLE: !Ref ListingsTable
  
  # DynamoDB Tables
  ListingsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: listing_id
          AttributeType: S
  
  # S3 Bucket
  ImagesBucket:
    Type: AWS::S3::Bucket
    Properties:
      LifecycleConfiguration:
        Rules:
          - ExpirationInDays: 90
```

**Deployment**:
```bash
sam build
sam deploy --guided
```

---

### 6.2 CI/CD Pipeline

```yaml
Tool: GitHub Actions

Workflow:
  1. Code push to main branch
  2. Run tests (Jest, pytest)
  3. Build Lambda packages
  4. SAM deploy to dev environment
  5. Run integration tests
  6. Manual approval gate
  7. SAM deploy to production
  8. Smoke tests

Environments:
  - dev: Minimal resources
  - prod: Full capacity, multi-AZ
```

---

## 7. Cost Estimation (MVP)

```yaml
Monthly AWS Costs (Estimated for 500 users):

Cognito: $0 (50,000 MAU free tier)

API Gateway: $15
  - 500K requests/month
  - $3.50 per million requests

Lambda: $25
  - 1M invocations
  - 500ms avg duration, 512MB memory
  - ~$20 compute + $5 requests

DynamoDB: $10
  - On-demand pricing
  - 100K reads, 50K writes/month

S3: $5
  - 100 GB storage
  - 10K PUT, 50K GET

Rekognition: $40
  - 10K custom label inferences
  - $4 per 1000

Bedrock (Claude Haiku): $5
  - 20M tokens (~40K conversations)
  - $0.25 per 1M tokens

CloudWatch: $5
  - Logs and metrics

SNS: $5
  - 10K SMS notifications

TOTAL: ~$110/month

Note: Will scale with usage. On-demand pricing auto-adjusts.
```

---

## 8. MVP Limitations & Future Enhancements

### Current MVP Scope
✅ Phone authentication  
✅ Image upload & AI grading  
✅ Listing marketplace  
✅ Basic chatbot  
✅ Simple booking  

### Post-MVP Enhancements
🔲 Payment escrow (Razorpay integration)  
🔲 Logistics tracking (Google Maps API)  
🔲 Advanced matching algorithm  
🔲 Multi-region deployment  
🔲 Real-time notifications (WebSocket)  
🔲 Farmer credit scoring  
🔲 Carbon credit calculation  

---

## 9. Disaster Recovery & Backup

```yaml
Backup Strategy:
  DynamoDB:
    - Point-in-time recovery (PITR): Enabled
    - On-demand backups: Weekly
  
  S3:
    - Versioning: Disabled (MVP)
    - Cross-region replication: No (cost)
  
  Lambda:
    - Code versioned in Git
    - Redeployable anytime

Recovery:
  RTO: 2 hours
  RPO: 24 hours (DynamoDB PITR)
```

---

## Document Version

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-02-16 | Initial MVP design for hackathon |

**Status**: Ready for ideation/demo submission
