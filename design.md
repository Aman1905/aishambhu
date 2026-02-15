# Agri-Chakra: System Design Document

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Architecture Design](#2-architecture-design)
3. [AWS Infrastructure Design](#3-aws-infrastructure-design)
4. [AI/ML Pipeline Design](#4-aiml-pipeline-design)
5. [Database Design](#5-database-design)
6. [API Design](#6-api-design)
7. [Security Architecture](#7-security-architecture)
8. [Deployment Architecture](#8-deployment-architecture)
9. [Monitoring and Observability](#9-monitoring-and-observability)
10. [Disaster Recovery](#10-disaster-recovery)

---

## 1. System Overview

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER INTERFACES                          │
├──────────────┬──────────────┬──────────────┬───────────────────┤
│ Mobile App   │  Web Portal  │ Admin Panel  │ WhatsApp Bot      │
│ (Farmers)    │  (Buyers)    │              │                   │
└──────┬───────┴──────┬───────┴──────┬───────┴────────┬──────────┘
       │              │              │                │
       └──────────────┴──────────────┴────────────────┘
                      │
              ┌───────▼────────┐
              │  API Gateway   │
              │  (AWS)         │
              └───────┬────────┘
                      │
       ┌──────────────┼──────────────┐
       │              │              │
┌──────▼─────┐ ┌─────▼──────┐ ┌────▼─────────┐
│ User Mgmt  │ │ Marketplace│ │ AI Grading   │
│ Service    │ │ Service    │ │ Service      │
└──────┬─────┘ └─────┬──────┘ └────┬─────────┘
       │              │              │
       │      ┌───────▼──────┐       │
       │      │ Aggregation  │       │
       │      │ Service      │       │
       │      └───────┬──────┘       │
       │              │              │
┌──────▼──────────────▼──────────────▼─────────┐
│         DATA LAYER (RDS + DynamoDB + S3)      │
└───────────────────────────────────────────────┘
```

### 1.2 Design Principles

- **Microservices Architecture**: Loosely coupled services for scalability
- **Event-Driven**: Asynchronous processing for resilience
- **Cloud-Native**: Serverless and managed services for cost optimization
- **API-First**: All functionality exposed through well-defined APIs
- **Mobile-First**: Optimized for smartphone users with limited bandwidth
- **Offline-Capable**: Local storage with sync for connectivity gaps
- **Multi-Tenancy**: Shared infrastructure with logical data isolation

---

## 2. Architecture Design

### 2.1 Microservices Architecture

#### Core Services

```
┌──────────────────────────────────────────────────────────────┐
│                    MICROSERVICES LAYER                        │
├────────────────┬────────────────┬──────────────┬─────────────┤
│ User Service   │ Listing        │ Transaction  │ Analytics   │
│                │ Service        │ Service      │ Service     │
│ - Registration │ - Create/Edit  │ - Booking    │ - Dashboards│
│ - Auth         │ - Search       │ - Payment    │ - Reports   │
│ - Profile      │ - Matching     │ - Escrow     │ - Insights  │
│ - KYC          │ - Discovery    │ - Settlement │ - Metrics   │
└────────────────┴────────────────┴──────────────┴─────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    AI/ML SERVICES LAYER                       │
├────────────────┬────────────────┬──────────────┬─────────────┤
│ Grading        │ Pricing        │ Fraud        │ Forecasting │
│ Service        │ Service        │ Detection    │ Service     │
│                │                │ Service      │             │
│ - Image Class. │ - Price Pred.  │ - Anomaly    │ - Demand    │
│ - Quality Grad.│ - Market       │ - Risk Score │ - Supply    │
│ - Quantity Est.│   Analysis     │ - Validation │ - Trends    │
└────────────────┴────────────────┴──────────────┴─────────────┘

┌──────────────────────────────────────────────────────────────┐
│                  SUPPORTING SERVICES LAYER                    │
├────────────────┬────────────────┬──────────────┬─────────────┤
│ Notification   │ Logistics      │ Aggregation  │ Content     │
│ Service        │ Service        │ Service      │ Service     │
│                │                │              │             │
│ - SMS/WhatsApp │ - Route Plan   │ - Clustering │ - Knowledge │
│ - Email        │ - Tracking     │ - Batch Mgmt │ - Education │
│ - Push Notif.  │ - Scheduling   │ - Collection │ - Videos    │
└────────────────┴────────────────┴──────────────┴─────────────┘
```

### 2.2 Service Communication Patterns

#### Synchronous Communication
- **API Gateway → Services**: REST over HTTPS
- **Inter-Service Calls**: gRPC for low latency
- **Client → API**: REST/GraphQL

#### Asynchronous Communication
- **Event Publishing**: Amazon EventBridge
- **Message Queuing**: Amazon SQS for reliable delivery
- **Pub/Sub**: Amazon SNS for notifications
- **Stream Processing**: Amazon Kinesis for real-time analytics

### 2.3 Data Flow Architecture

```
┌──────────────┐
│ Farmer App   │
└──────┬───────┘
       │ 1. Upload Images
       ▼
┌──────────────────────┐
│ API Gateway          │
│ + CloudFront (CDN)   │
└──────┬───────────────┘
       │ 2. Store in S3
       ▼
┌──────────────────────┐      ┌────────────────────┐
│ Amazon S3            │─────▶│ Lambda Trigger     │
│ (Image Storage)      │      │ (Pre-processing)   │
└──────────────────────┘      └────────┬───────────┘
                                       │ 3. Invoke AI
                                       ▼
                              ┌────────────────────┐
                              │ SageMaker Endpoint │
                              │ (Grading Model)    │
                              └────────┬───────────┘
                                       │ 4. Results
                                       ▼
                              ┌────────────────────┐
                              │ DynamoDB           │
                              │ (Grading Results)  │
                              └────────┬───────────┘
                                       │ 5. Publish Event
                                       ▼
                              ┌────────────────────┐
                              │ EventBridge        │
                              └────────┬───────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
           ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
           │ Update Listing │ │ Send Notification│ │ Update Analytics│
           └────────────────┘ └────────────────┘ └────────────────┘
```

---

## 3. AWS Infrastructure Design

### 3.1 Network Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS CLOUD                                │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    PRODUCTION VPC                           │ │
│  │                  (10.0.0.0/16)                             │ │
│  │                                                             │ │
│  │  ┌───────────────────┐  ┌───────────────────┐             │ │
│  │  │  Public Subnet-1  │  │  Public Subnet-2  │             │ │
│  │  │  (10.0.1.0/24)    │  │  (10.0.2.0/24)    │             │ │
│  │  │  us-east-1a       │  │  us-east-1b       │             │ │
│  │  │                   │  │                   │             │ │
│  │  │  - NAT Gateway    │  │  - NAT Gateway    │             │ │
│  │  │  - ALB            │  │  - ALB            │             │ │
│  │  └───────────────────┘  └───────────────────┘             │ │
│  │                                                             │ │
│  │  ┌───────────────────┐  ┌───────────────────┐             │ │
│  │  │ Private Subnet-1  │  │ Private Subnet-2  │             │ │
│  │  │  (10.0.10.0/24)   │  │  (10.0.11.0/24)   │             │ │
│  │  │  us-east-1a       │  │  us-east-1b       │             │ │
│  │  │                   │  │                   │             │ │
│  │  │  - ECS Tasks      │  │  - ECS Tasks      │             │ │
│  │  │  - Lambda         │  │  - Lambda         │             │ │
│  │  └───────────────────┘  └───────────────────┘             │ │
│  │                                                             │ │
│  │  ┌───────────────────┐  ┌───────────────────┐             │ │
│  │  │   Data Subnet-1   │  │   Data Subnet-2   │             │ │
│  │  │  (10.0.20.0/24)   │  │  (10.0.21.0/24)   │             │ │
│  │  │  us-east-1a       │  │  us-east-1b       │             │ │
│  │  │                   │  │                   │             │ │
│  │  │  - RDS Primary    │  │  - RDS Standby    │             │ │
│  │  │  - ElastiCache    │  │  - ElastiCache    │             │ │
│  │  └───────────────────┘  └───────────────────┘             │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Compute Infrastructure

#### ECS/Fargate for Microservices
```yaml
Service Configuration:
  - Cluster: agri-chakra-prod
  - Launch Type: Fargate (serverless containers)
  - Task Definition:
      CPU: 1024 (1 vCPU)
      Memory: 2048 MB
      Container Image: ECR (Amazon Elastic Container Registry)
  - Auto Scaling:
      Min Tasks: 2
      Max Tasks: 20
      Target CPU: 70%
      Target Memory: 80%
  - Load Balancing: Application Load Balancer
  - Health Checks: /health endpoint
```

#### Lambda Functions
```yaml
Function Categories:
  1. API Handlers:
     - Runtime: Node.js 20.x / Python 3.12
     - Memory: 512-1024 MB
     - Timeout: 30s
     - Concurrency: 100-1000
  
  2. Event Processors:
     - Runtime: Python 3.12
     - Memory: 1024-3072 MB
     - Timeout: 5 minutes
     - Trigger: SQS/EventBridge
  
  3. ML Inference:
     - Runtime: Python 3.12 + Container
     - Memory: 3008 MB (max)
     - Timeout: 15 minutes
     - Ephemeral Storage: 10 GB
```

### 3.3 Storage Architecture

#### Amazon S3 Buckets
```
agri-chakra-images-prod/
├── raw-uploads/          # Original farmer uploads
│   ├── {user_id}/
│   │   └── {timestamp}-{image_id}.jpg
│   └── lifecycle: 90 days → Glacier
├── processed/            # AI-processed images
│   ├── thumbnails/
│   ├── annotated/
│   └── lifecycle: 365 days → Glacier
├── model-artifacts/      # ML model files
│   └── versions/
├── exports/              # Batch data exports
└── logs/                 # Application logs

Configuration:
  - Versioning: Enabled on critical buckets
  - Encryption: SSE-KMS (AWS managed keys)
  - Access: Private (IAM + bucket policies)
  - CORS: Enabled for web/mobile uploads
  - Transfer Acceleration: Enabled
```

#### Amazon RDS (PostgreSQL)
```yaml
Configuration:
  Engine: PostgreSQL 15.4
  Instance Class: db.r6g.xlarge (4 vCPU, 32 GB RAM)
  Storage:
    Type: gp3 (SSD)
    Size: 500 GB
    IOPS: 12,000
    Throughput: 500 MB/s
    Auto-Scaling: Up to 2 TB
  
  High Availability:
    Multi-AZ: Yes
    Read Replicas: 2 (for analytics)
    Backup Retention: 30 days
    Automated Backups: Daily at 03:00 UTC
  
  Performance:
    Connection Pooling: RDS Proxy
    Query Cache: Enabled
    Monitoring: Enhanced monitoring (1-minute)
```

#### Amazon DynamoDB
```yaml
Tables:
  1. ListingsTable:
     - Partition Key: listing_id (String)
     - Sort Key: created_at (Number)
     - GSI: user_id-status-index, location-residue_type-index
     - Capacity: On-Demand (auto-scaling)
     - TTL: enabled (auto-expire old listings)
     - Point-in-Time Recovery: Enabled
  
  2. GradingResultsTable:
     - Partition Key: image_id (String)
     - Attributes: grade, confidence, metadata
     - Capacity: On-Demand
  
  3. UserSessionsTable:
     - Partition Key: session_id (String)
     - TTL: 24 hours
     - Capacity: On-Demand
```

#### Amazon ElastiCache (Redis)
```yaml
Configuration:
  Engine: Redis 7.0
  Node Type: cache.r6g.large (2 vCPU, 13 GB RAM)
  Cluster Mode: Enabled (3 shards, 2 replicas each)
  
  Use Cases:
    - API response caching (5-minute TTL)
    - User session storage
    - Rate limiting counters
    - Real-time leaderboards
    - Pub/sub for notifications
  
  High Availability:
    Multi-AZ: Yes
    Automatic Failover: Enabled
    Backup: Daily snapshots (7 days retention)
```

---

## 4. AI/ML Pipeline Design

### 4.1 Model Training Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                   ML TRAINING PIPELINE                           │
└─────────────────────────────────────────────────────────────────┘

Step 1: Data Collection
┌──────────────────┐
│ S3 Data Lake     │
│ - Raw Images     │──┐
│ - Labels         │  │
│ - Metadata       │  │
└──────────────────┘  │
                       │
Step 2: Data Preparation  │
┌──────────────────┐  │
│ AWS Glue ETL     │◀─┘
│ - Cleaning       │
│ - Augmentation   │──┐
│ - Feature Eng.   │  │
└──────────────────┘  │
                       │
Step 3: Training       │
┌──────────────────┐  │
│ SageMaker        │◀─┘
│ Training Job     │
│ - GPU Instances  │──┐
│ - Distributed    │  │
└──────────────────┘  │
                       │
Step 4: Evaluation     │
┌──────────────────┐  │
│ Model Validation │◀─┘
│ - Accuracy       │
│ - Confusion Mat. │──┐
│ - A/B Testing    │  │
└──────────────────┘  │
                       │
Step 5: Deployment     │
┌──────────────────┐  │
│ SageMaker        │◀─┘
│ Endpoint         │
│ - Auto-scaling   │
│ - Monitoring     │
└──────────────────┘
```

### 4.2 SageMaker Model Configuration

#### Residue Classification Model
```python
# Model: ResNet-50 Transfer Learning
{
  "training_instance": "ml.p3.2xlarge",  # Tesla V100 GPU
  "training_duration": "4-6 hours",
  "dataset_size": "50,000 images",
  "batch_size": 64,
  "epochs": 50,
  "optimizer": "Adam",
  "learning_rate": 0.0001,
  
  "inference_instance": "ml.c5.xlarge",
  "inference_mode": "real-time",
  "latency_target": "< 2 seconds",
  "throughput": "100 requests/second"
}
```

#### Quality Grading Model
```python
# Model: Multi-task CNN + XGBoost
{
  "training_instance": "ml.p3.8xlarge",  # 4x V100
  "training_duration": "8-12 hours",
  "features": [
    "visual_features (CNN)",
    "moisture_estimation",
    "contamination_score",
    "color_histogram",
    "texture_analysis",
    "metadata (location, season)"
  ],
  
  "output": {
    "grade": ["A", "B", "C"],
    "confidence": "0.0-1.0",
    "quality_scores": {
      "moisture": "percentage",
      "purity": "percentage",
      "calorific_value": "kcal/kg"
    }
  }
}
```

#### Pricing Model
```python
# Model: LightGBM Regressor
{
  "training_instance": "ml.m5.2xlarge",
  "training_frequency": "Daily",
  "features": [
    "quality_grade",
    "residue_type",
    "quantity",
    "location (lat, long)",
    "season",
    "historical_prices (7/30/90 days)",
    "demand_supply_ratio",
    "logistics_cost"
  ],
  
  "output": {
    "fair_price_min": "INR/ton",
    "fair_price_max": "INR/ton",
    "market_trend": "rising/stable/falling"
  }
}
```

### 4.3 Model Versioning and A/B Testing

```yaml
SageMaker Model Registry:
  - Models: Grading_v1.0, Grading_v1.1, Grading_v2.0
  - Status: Approved, PendingApproval, Rejected
  - Metadata: Accuracy, F1-score, Inference latency
  
A/B Testing Strategy:
  - Traffic Split: 90% production model, 10% challenger
  - Metrics: Accuracy, latency, user satisfaction
  - Rollback: Automatic if challenger < 85% accuracy
  - Duration: 7 days before full rollout
```

### 4.4 Real-time Inference Architecture

```
┌──────────────┐
│ Mobile App   │
└──────┬───────┘
       │ POST /grade-residue
       ▼
┌───────────────────┐
│ API Gateway       │
└──────┬────────────┘
       │ Invoke Lambda
       ▼
┌───────────────────┐       ┌─────────────────────┐
│ Pre-process       │──────▶│ S3 Image Storage    │
│ Lambda            │       └─────────────────────┘
│ - Resize          │
│ - Normalize       │
└──────┬────────────┘
       │ Invoke SageMaker
       ▼
┌───────────────────┐       ┌─────────────────────┐
│ SageMaker         │◀──────│ Model Artifacts (S3)│
│ Endpoint          │       └─────────────────────┘
│ - Multi-model     │
│ - Auto-scaling    │
└──────┬────────────┘
       │ Results
       ▼
┌───────────────────┐       ┌─────────────────────┐
│ Post-process      │──────▶│ DynamoDB            │
│ Lambda            │       │ (Store Results)     │
│ - Format          │       └─────────────────────┘
│ - Enrich          │
└──────┬────────────┘
       │ Response
       ▼
┌───────────────────┐
│ Mobile App        │
│ (Display Grade)   │
└───────────────────┘
```

### 4.5 Batch Processing Pipeline

```yaml
AWS Batch Configuration:
  Use Case: Bulk grading (100+ images)
  
  Compute Environment:
    Type: Fargate Spot (cost-optimized)
    vCPU: 4-16
    Memory: 8-32 GB
  
  Job Queue: agri-chakra-batch-grading
  
  Job Definition:
    Image: ECR image with TensorFlow/PyTorch
    Command: python batch_grade.py --input s3://... --output s3://...
    Environment:
      - MODEL_ENDPOINT
      - BATCH_SIZE=50
    
  Workflow:
    1. Trigger: S3 event (new CSV with image URLs)
    2. Step Functions orchestrates:
       - Download images from S3
       - Process in batches of 50
       - Invoke SageMaker batch transform
       - Aggregate results
       - Update DynamoDB
       - Send notification
```

---

## 5. Database Design

### 5.1 Relational Database Schema (PostgreSQL)

#### Users Table
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_type VARCHAR(20) NOT NULL CHECK (user_type IN ('farmer', 'buyer', 'aggregator', 'admin')),
    phone_number VARCHAR(15) UNIQUE NOT NULL,
    aadhaar_hash VARCHAR(64),  -- Hashed, not plaintext
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    location GEOGRAPHY(POINT, 4326),  -- PostGIS extension
    address JSONB,
    kyc_status VARCHAR(20) DEFAULT 'pending',
    kyc_documents JSONB,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    last_login TIMESTAMP,
    profile_data JSONB,  -- Flexible attributes
    
    INDEX idx_phone (phone_number),
    INDEX idx_location USING GIST (location),
    INDEX idx_user_type (user_type, is_verified)
);

CREATE TABLE farmer_profiles (
    farmer_id UUID PRIMARY KEY REFERENCES users(user_id),
    land_holdings JSONB[],  -- Array of {area, location, crop_type}
    preferred_crops VARCHAR[],
    bank_account JSONB,  -- Encrypted
    upi_id VARCHAR(100),
    total_earnings DECIMAL(12,2) DEFAULT 0,
    rating DECIMAL(3,2),
    transactions_count INT DEFAULT 0
);

CREATE TABLE buyer_profiles (
    buyer_id UUID PRIMARY KEY REFERENCES users(user_id),
    company_name VARCHAR(200) NOT NULL,
    gst_number VARCHAR(15) UNIQUE NOT NULL,
    business_type VARCHAR(50),
    annual_requirement JSONB,  -- {residue_type: quantity_tons}
    procurement_locations GEOGRAPHY(POINT, 4326)[],
    credit_limit DECIMAL(12,2),
    rating DECIMAL(3,2),
    transactions_count INT DEFAULT 0
);
```

#### Listings Table
```sql
CREATE TABLE listings (
    listing_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    farmer_id UUID NOT NULL REFERENCES users(user_id),
    residue_type VARCHAR(50) NOT NULL,  -- rice_straw, bagasse, etc.
    quantity_tons DECIMAL(8,2) NOT NULL,
    quality_grade VARCHAR(1) CHECK (quality_grade IN ('A', 'B', 'C')),
    grade_confidence DECIMAL(5,4),
    price_per_ton DECIMAL(10,2),
    price_negotiable BOOLEAN DEFAULT TRUE,
    
    location GEOGRAPHY(POINT, 4326) NOT NULL,
    address TEXT,
    
    images JSONB,  -- Array of S3 URLs
    grading_data JSONB,  -- AI analysis results
    
    available_from DATE NOT NULL,
    available_until DATE NOT NULL,
    
    status VARCHAR(20) DEFAULT 'active' 
        CHECK (status IN ('draft', 'active', 'booked', 'sold', 'expired', 'cancelled')),
    
    views_count INT DEFAULT 0,
    bookmarks_count INT DEFAULT 0,
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_status (status) WHERE status = 'active',
    INDEX idx_location USING GIST (location),
    INDEX idx_residue_type (residue_type, quality_grade),
    INDEX idx_farmer (farmer_id, status)
);

CREATE TABLE grading_history (
    grading_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id UUID REFERENCES listings(listing_id),
    image_urls JSONB NOT NULL,
    
    model_version VARCHAR(20),
    residue_type_detected VARCHAR(50),
    quality_grade VARCHAR(1),
    confidence_score DECIMAL(5,4),
    
    analysis_results JSONB,  -- Detailed breakdown
    processing_time_ms INT,
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_listing (listing_id)
);
```

#### Transactions Table
```sql
CREATE TABLE transactions (
    transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id UUID REFERENCES listings(listing_id),
    buyer_id UUID REFERENCES users(user_id),
    farmer_id UUID REFERENCES users(user_id),
    
    quantity_tons DECIMAL(8,2) NOT NULL,
    price_per_ton DECIMAL(10,2) NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    
    platform_fee DECIMAL(10,2),
    logistics_cost DECIMAL(10,2),
    
    status VARCHAR(20) DEFAULT 'pending'
        CHECK (status IN ('pending', 'confirmed', 'in_transit', 'delivered', 'completed', 'cancelled', 'disputed')),
    
    payment_status VARCHAR(20) DEFAULT 'pending'
        CHECK (payment_status IN ('pending', 'advance_paid', 'escrow', 'released', 'refunded')),
    
    payment_details JSONB,
    
    booking_date TIMESTAMP DEFAULT NOW(),
    expected_delivery DATE,
    actual_delivery TIMESTAMP,
    
    pickup_location GEOGRAPHY(POINT, 4326),
    delivery_location GEOGRAPHY(POINT, 4326),
    
    logistics_partner VARCHAR(100),
    tracking_id VARCHAR(100),
    
    quality_verification JSONB,  -- Post-delivery inspection
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_buyer (buyer_id, status),
    INDEX idx_farmer (farmer_id, status),
    INDEX idx_status (status, payment_status)
);

CREATE TABLE payments (
    payment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID REFERENCES transactions(transaction_id),
    
    payment_type VARCHAR(20) CHECK (payment_type IN ('advance', 'final', 'refund')),
    amount DECIMAL(12,2) NOT NULL,
    
    payment_method VARCHAR(20),  -- upi, card, netbanking
    payment_gateway VARCHAR(50),
    gateway_transaction_id VARCHAR(100),
    
    status VARCHAR(20) CHECK (status IN ('initiated', 'success', 'failed', 'refunded')),
    
    initiated_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    
    metadata JSONB
);
```

### 5.2 NoSQL Data Model (DynamoDB)

#### Listings Cache (Real-time Search)
```json
{
  "TableName": "ListingsCache",
  "KeySchema": {
    "PartitionKey": "listing_id",
    "SortKey": "created_at"
  },
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "location-residue-index",
      "PartitionKey": "geohash_6",  // 6-character geohash
      "SortKey": "residue_type#quality_grade",
      "ProjectedAttributes": "ALL"
    },
    {
      "IndexName": "status-updated-index",
      "PartitionKey": "status",
      "SortKey": "updated_at",
      "ProjectedAttributes": "KEYS_ONLY"
    }
  ],
  
  "SampleRecord": {
    "listing_id": "550e8400-e29b-41d4-a716-446655440000",
    "created_at": 1704067200000,
    "farmer_id": "farmer-12345",
    "residue_type": "rice_straw",
    "quality_grade": "A",
    "quantity_tons": 15.5,
    "price_per_ton": 3500,
    "location": {
      "lat": 30.7333,
      "long": 76.7794,
      "geohash_6": "ttnv6u"
    },
    "status": "active",
    "images": ["s3://..."],
    "updated_at": 1704153600000,
    "ttl": 1706745600  // Auto-expire after 30 days
  }
}
```

#### User Sessions
```json
{
  "TableName": "UserSessions",
  "KeySchema": {
    "PartitionKey": "session_id"
  },
  
  "SampleRecord": {
    "session_id": "sess_abc123xyz",
    "user_id": "user-789",
    "device_info": {
      "platform": "android",
      "version": "1.2.0",
      "device_id": "unique-device-id"
    },
    "ip_address": "103.x.x.x",
    "created_at": 1704067200000,
    "last_activity": 1704070800000,
    "ttl": 1704153600  // 24 hours expiry
  }
}
```

### 5.3 Data Partitioning Strategy

```yaml
Partitioning Approach:

1. Horizontal Partitioning (Sharding):
   - Users: Hash-based on user_id
   - Listings: Range-based on created_at (yearly partitions)
   - Transactions: Range-based on booking_date (quarterly)

2. Vertical Partitioning:
   - Hot data (active listings): DynamoDB
   - Cold data (historical): RDS + S3 archival
   - Analytics data: Separate read replicas

3. Geographic Partitioning:
   - Primary Region: ap-south-1 (Mumbai)
   - Read Replicas: ap-southeast-1 (Singapore)
   - Data Residency: Comply with local regulations
```

---

## 6. API Design

### 6.1 RESTful API Structure

```
Base URL: https://api.agri-chakra.com/v1

Authentication: JWT Bearer Token
Rate Limiting: 100 requests/minute (authenticated)
```

#### Core Endpoints

```yaml
Authentication & Users:
  POST   /auth/register          # User registration
  POST   /auth/verify-otp        # OTP verification
  POST   /auth/login             # Login with phone
  POST   /auth/refresh           # Refresh access token
  GET    /users/profile          # Get user profile
  PUT    /users/profile          # Update profile
  POST   /users/kyc              # Submit KYC documents

Listings:
  GET    /listings               # Search/filter listings
         ?residue_type=rice_straw
         &quality_grade=A
         &lat=30.73&long=76.77&radius=50
         &min_quantity=10&max_price=4000
         &page=1&limit=20
  
  POST   /listings               # Create new listing
  GET    /listings/{id}          # Get listing details
  PUT    /listings/{id}          # Update listing
  DELETE /listings/{id}          # Delete/cancel listing
  
  POST   /listings/{id}/bookmark # Bookmark listing
  GET    /listings/my-listings   # Farmer's own listings

Grading (AI):
  POST   /grading/upload         # Upload images for grading
         # Multipart form data
         # Returns: grading_job_id
  
  GET    /grading/{job_id}       # Get grading status/results
  POST   /grading/batch          # Batch grading (CSV upload)

Transactions:
  POST   /transactions           # Create booking
  GET    /transactions/{id}      # Get transaction details
  PUT    /transactions/{id}      # Update status
  GET    /transactions/my-purchases  # Buyer's transactions
  GET    /transactions/my-sales      # Farmer's transactions
  
  POST   /transactions/{id}/confirm       # Confirm booking
  POST   /transactions/{id}/cancel        # Cancel transaction
  POST   /transactions/{id}/quality-check # Post-delivery verification

Payments:
  POST   /payments/initiate      # Initiate payment
  POST   /payments/webhook       # Payment gateway callback
  GET    /payments/{id}/status   # Payment status

Pricing:
  GET    /pricing/estimate       # Get price estimate
         ?residue_type=bagasse&quality_grade=B&quantity=20&location=...
  
  GET    /pricing/trends         # Historical price trends

Analytics:
  GET    /analytics/dashboard    # User dashboard metrics
  GET    /analytics/market       # Market overview
  GET    /analytics/impact       # Environmental impact

Notifications:
  GET    /notifications          # Get notifications
  PUT    /notifications/{id}/read # Mark as read
  POST   /notifications/preferences # Update notification settings
```

### 6.2 GraphQL API (Optional, for complex queries)

```graphql
type Query {
  listings(
    filters: ListingFilters
    location: LocationInput
    pagination: PaginationInput
  ): ListingConnection!
  
  listing(id: ID!): Listing
  
  transactions(
    userId: ID!
    role: UserRole!
    status: TransactionStatus
  ): [Transaction!]!
  
  marketAnalytics(
    region: String!
    residueType: ResidueType
    timeRange: TimeRange!
  ): MarketAnalytics!
}

type Mutation {
  createListing(input: CreateListingInput!): Listing!
  updateListing(id: ID!, input: UpdateListingInput!): Listing!
  
  gradeImages(images: [Upload!]!): GradingJob!
  
  bookListing(
    listingId: ID!
    quantity: Float!
    deliveryDate: Date!
  ): Transaction!
  
  initiatePayment(
    transactionId: ID!
    amount: Float!
    method: PaymentMethod!
  ): Payment!
}

type Subscription {
  gradingJobUpdated(jobId: ID!): GradingJob!
  transactionStatusChanged(transactionId: ID!): Transaction!
  newMatchingListing(filters: ListingFilters!): Listing!
}
```

### 6.3 API Gateway Configuration

```yaml
AWS API Gateway Setup:

Stage: Production
  - Deployment: Blue-Green
  - Throttling: 10,000 requests/second (burst)
  - Cache: 5 minutes TTL for GET requests
  
Authorization:
  - Cognito User Pool integration
  - API Key for mobile apps
  - IAM roles for service-to-service

CORS Configuration:
  - Allowed Origins: https://agri-chakra.com, mobile-app://
  - Allowed Methods: GET, POST, PUT, DELETE, OPTIONS
  - Allowed Headers: Content-Type, Authorization
  - Max Age: 3600

Request Validation:
  - JSON Schema validation
  - Request size limit: 10 MB (for image uploads)
  - Query parameter validation

Response Transformation:
  - Error normalization
  - Pagination metadata injection
  - HATEOAS links (optional)

Logging:
  - CloudWatch Logs: Full request/response
  - X-Ray: Distributed tracing
  - Metrics: Latency, 4xx, 5xx errors
```

### 6.4 Webhooks

```yaml
Webhook Events:

1. Listing Events:
   - listing.created
   - listing.updated
   - listing.expired
   - listing.booked

2. Transaction Events:
   - transaction.created
   - transaction.confirmed
   - transaction.in_transit
   - transaction.delivered
   - transaction.completed
   - transaction.cancelled

3. Payment Events:
   - payment.initiated
   - payment.success
   - payment.failed
   - payment.refunded

4. Grading Events:
   - grading.job_started
   - grading.job_completed
   - grading.job_failed

Webhook Payload Format:
{
  "event_type": "transaction.completed",
  "event_id": "evt_abc123",
  "timestamp": "2025-02-15T10:30:00Z",
  "data": {
    "transaction_id": "txn_xyz789",
    "farmer_id": "farmer-123",
    "buyer_id": "buyer-456",
    ...
  },
  "signature": "HMAC-SHA256 signature"
}

Security:
  - HMAC signature verification
  - Retry mechanism (3 attempts with exponential backoff)
  - Delivery guarantee: At-least-once
```

---

## 7. Security Architecture

### 7.1 Authentication and Authorization

```
┌────────────────────────────────────────────────────────────┐
│              AUTHENTICATION FLOW                            │
└────────────────────────────────────────────────────────────┘

1. User Registration:
   ┌──────────────┐
   │ Mobile App   │
   └──────┬───────┘
          │ POST /auth/register (phone_number)
          ▼
   ┌──────────────────┐
   │ API Gateway      │
   └──────┬───────────┘
          │
          ▼
   ┌──────────────────┐      ┌─────────────────┐
   │ Lambda           │─────▶│ Amazon Cognito  │
   │ (Auth Service)   │      │ User Pool       │
   └──────┬───────────┘      └─────────────────┘
          │ Send OTP via SNS
          ▼
   ┌──────────────────┐
   │ User Phone       │
   │ (OTP: 123456)    │
   └──────────────────┘

2. OTP Verification:
   ┌──────────────┐
   │ Mobile App   │─── POST /auth/verify-otp (phone, otp)
   └──────────────┘
          │
          ▼
   ┌──────────────────┐
   │ Cognito          │─── Issues JWT Tokens
   │                  │    - Access Token (1 hour)
   │                  │    - Refresh Token (30 days)
   └──────────────────┘

3. API Request:
   ┌──────────────┐
   │ Mobile App   │─── GET /listings
   │              │    Header: Authorization: Bearer <access_token>
   └──────────────┘
          │
          ▼
   ┌──────────────────┐
   │ API Gateway      │─── Cognito Authorizer validates JWT
   │                  │    Extracts user_id, role
   └──────┬───────────┘
          │ Authorized
          ▼
   ┌──────────────────┐
   │ Lambda/ECS       │─── Process request
   │ (Business Logic) │    Access user context
   └──────────────────┘
```

### 7.2 Data Encryption

```yaml
Encryption at Rest:
  S3:
    - SSE-KMS with customer managed keys
    - Key rotation: Annual
    - Bucket versioning: Enabled
  
  RDS:
    - TDE (Transparent Data Encryption)
    - Encrypted backups
    - Encrypted snapshots
  
  DynamoDB:
    - AWS managed encryption
    - Point-in-time recovery encrypted
  
  Secrets:
    - AWS Secrets Manager for credentials
    - Parameter Store for configuration
    - Automatic rotation for DB passwords

Encryption in Transit:
  - TLS 1.3 for all API endpoints
  - Certificate: AWS Certificate Manager
  - HTTPS only (HTTP → HTTPS redirect)
  - HSTS header enabled
  - Perfect Forward Secrecy

Sensitive Data Handling:
  - PII (Phone, Aadhaar): Hashed with bcrypt/Argon2
  - Bank Details: AES-256 encryption
  - Images: No metadata stripping (EXIF preserved for analysis)
  - Logs: Scrubbed of sensitive data
```

### 7.3 Network Security

```yaml
VPC Security:
  - Private subnets for all compute/data layers
  - NAT Gateways for outbound internet (software updates)
  - No public IPs on ECS tasks or Lambda
  - VPC endpoints for AWS services (S3, DynamoDB, Secrets Manager)

Security Groups:
  ALB-SG:
    Inbound: 443 (HTTPS) from 0.0.0.0/0
    Outbound: 8080 to ECS-SG
  
  ECS-SG:
    Inbound: 8080 from ALB-SG
    Outbound: 5432 to RDS-SG, 443 to VPC endpoints
  
  RDS-SG:
    Inbound: 5432 from ECS-SG, Lambda-SG
    Outbound: None
  
  Lambda-SG:
    Inbound: None
    Outbound: 5432 to RDS-SG, 443 to internet (via NAT)

Network ACLs:
  - Stateless firewall rules
  - Block known malicious IPs
  - Rate limiting at network level

WAF (Web Application Firewall):
  Rules:
    - SQL injection protection
    - XSS protection
    - Rate-based rule: 100 requests/5 minutes per IP
    - Geo-blocking: India + approved countries only
    - Custom rule: Block suspicious User-Agents
```

### 7.4 Application Security

```yaml
Input Validation:
  - JSON schema validation (API Gateway)
  - File upload restrictions:
      - Allowed: .jpg, .jpeg, .png
      - Max size: 10 MB per image
      - Content-Type verification
      - Virus scanning (ClamAV Lambda)
  - SQL injection prevention: Parameterized queries
  - NoSQL injection: Input sanitization

Authentication:
  - Multi-factor authentication for high-value transactions
  - Biometric (fingerprint/face) for payments
  - Device fingerprinting
  - Session timeout: 30 minutes inactivity
  - Concurrent session limit: 3 devices

Authorization:
  - Role-Based Access Control (RBAC):
      Farmer: Can create listings, view own transactions
      Buyer: Can book listings, view purchases
      Admin: Full access
      Aggregator: Manage collections, view region data
  
  - Resource-level permissions:
      Users can only modify their own listings/profile
      Verified KYC required for transactions > ₹50,000

Rate Limiting:
  - API Gateway: 100 requests/minute per user
  - Image upload: 10 uploads/hour per user
  - Payment initiation: 5 attempts/day
  - Failed login: Lock account after 5 attempts (15 min)

Fraud Detection:
  - ML-based anomaly detection:
      - Duplicate image detection (perceptual hash)
      - Fake listing patterns
      - Velocity checks (too many transactions)
      - Geographic anomalies
  - Manual review queue for flagged activities
```

### 7.5 Compliance and Privacy

```yaml
Data Privacy:
  - GDPR/DPDP Act compliance:
      - User consent for data collection
      - Right to access personal data
      - Right to deletion (soft delete, 90-day retention)
      - Data minimization
      - Privacy policy and terms acceptance
  
  - Aadhaar compliance:
      - No plaintext storage
      - SHA-256 hash only
      - Masked display (XXXX-XXXX-1234)
      - UIDAI guidelines adherence
  
  - Payment data:
      - PCI DSS compliance (Level 1)
      - Tokenization for card storage
      - No CVV storage
      - SAQ-D questionnaire

Audit Logging:
  - CloudTrail: All AWS API calls
  - Application logs:
      - User actions (login, listing creation, payments)
      - Admin actions (KYC approval, listing moderation)
      - Security events (failed login, permission denied)
  - Log retention: 1 year (compressed in S3 Glacier)
  - SIEM integration: Splunk/Sumo Logic

Vulnerability Management:
  - Dependency scanning: Snyk/Dependabot
  - Container scanning: AWS ECR vulnerability scan
  - SAST: SonarQube in CI/CD
  - DAST: OWASP ZAP automated scans
  - Penetration testing: Quarterly (third-party)
  - Bug bounty program: HackerOne (after launch)
```

---

## 8. Deployment Architecture

### 8.1 CI/CD Pipeline

```yaml
Source Control: GitHub

Pipeline Stages:

1. Code Commit (GitHub):
   ├─ Trigger: Push to main/develop branches
   └─ Webhook → AWS CodePipeline

2. Build Stage (AWS CodeBuild):
   ├─ Checkout code
   ├─ Install dependencies
   ├─ Run linters (ESLint, Pylint)
   ├─ Run unit tests (Jest, pytest)
   ├─ Code coverage (>80% threshold)
   ├─ SAST scan (SonarQube)
   ├─ Build Docker images
   └─ Push to Amazon ECR

3. Test Stage:
   ├─ Deploy to staging environment
   ├─ Integration tests (Postman/Newman)
   ├─ E2E tests (Cypress)
   ├─ Load tests (k6, 1000 VUs)
   ├─ Security scan (OWASP ZAP)
   └─ Approval gate (manual for production)

4. Deploy Stage:
   ├─ Blue-Green deployment
   ├─ Update ECS task definitions
   ├─ Update Lambda functions (versioned)
   ├─ Database migrations (Flyway/Alembic)
   ├─ Update API Gateway stages
   ├─ Smoke tests
   └─ Route traffic (10% → 50% → 100%)

5. Post-Deployment:
   ├─ Monitor CloudWatch metrics
   ├─ Check error rates (threshold: <1%)
   ├─ Latency checks (p95 < 500ms)
   └─ Rollback if failures detected

Rollback Strategy:
  - Automated rollback on critical errors
  - Keep previous 5 versions
  - Blue-Green allows instant rollback
  - Database migrations: Backward compatible
```

### 8.2 Infrastructure as Code

```yaml
Tool: Terraform

Repository Structure:
terraform/
├── modules/
│   ├── networking/         # VPC, subnets, route tables
│   ├── compute/            # ECS, Lambda
│   ├── storage/            # S3, RDS, DynamoDB
│   ├── ai-ml/              # SageMaker endpoints
│   ├── security/           # IAM, KMS, WAF
│   └── monitoring/         # CloudWatch, alarms
├── environments/
│   ├── dev/
│   ├── staging/
│   └── production/
└── main.tf

Deployment Process:
  1. terraform plan -out=tfplan
  2. Review plan (Git PR review)
  3. terraform apply tfplan
  4. State stored in S3 with DynamoDB locking

State Management:
  Backend: S3 bucket (versioned, encrypted)
  Locking: DynamoDB table
  Remote state sharing between modules
```

### 8.3 Environment Strategy

```yaml
Environments:

1. Development:
   - Purpose: Feature development
   - Instances: t3.small (low cost)
   - Database: Single AZ, no replicas
   - Auto-scaling: Disabled
   - Monitoring: Basic

2. Staging:
   - Purpose: Integration testing, UAT
   - Instances: Production-like (scaled down 50%)
   - Database: Multi-AZ, 1 read replica
   - Auto-scaling: Enabled
   - Monitoring: Full observability
   - Data: Anonymized production data

3. Production:
   - Purpose: Live traffic
   - Instances: c6g.xlarge (ARM Graviton2)
   - Database: Multi-AZ, 2 read replicas
   - Auto-scaling: Aggressive
   - Monitoring: Real-time alerts
   - Backup: Daily, 30-day retention
   - Blue-Green: Enabled

4. Disaster Recovery (DR):
   - Region: ap-southeast-1 (Singapore)
   - RTO: 4 hours
   - RPO: 15 minutes
   - Pilot light configuration
   - Automated failover testing: Monthly
```

---

## 9. Monitoring and Observability

### 9.1 Metrics and Dashboards

```yaml
CloudWatch Dashboards:

1. Application Health Dashboard:
   Metrics:
     - API Gateway: Request count, latency (p50, p95, p99), 4xx/5xx errors
     - Lambda: Invocations, duration, errors, throttles
     - ECS: CPU/Memory utilization, task count
     - RDS: Connections, read/write IOPS, latency
     - DynamoDB: Consumed capacity, throttled requests
   
   Widgets:
     - Line charts for trends (1h, 24h, 7d views)
     - Single-value metrics (current values)
     - Heatmaps for latency distribution

2. Business Metrics Dashboard:
   Metrics:
     - Total listings created (daily/weekly)
     - Active listings count
     - Transactions initiated, completed
     - GMV (Gross Merchandise Value)
     - User registrations, KYC completion rate
     - Average grading time
     - Platform revenue
   
   Source: Custom metrics from application

3. AI/ML Dashboard:
   Metrics:
     - SageMaker endpoint latency, invocations
     - Model accuracy (from validation set)
     - Grading throughput (images/hour)
     - Model drift detection
     - Feature importance changes
   
   Tools: SageMaker Model Monitor

4. Security Dashboard:
   Metrics:
     - WAF blocked requests
     - Failed authentication attempts
     - Suspicious activities flagged
     - KMS key usage
     - IAM policy changes
   
   Tools: CloudWatch + AWS Security Hub
```

### 9.2 Logging Strategy

```yaml
Centralized Logging:
  Tool: CloudWatch Logs + S3 (long-term storage)

Log Groups:
  - /aws/lambda/{function-name}
  - /ecs/agri-chakra/{service-name}
  - /api-gateway/production
  - /rds/postgresql/error
  - /application/business-events

Log Levels:
  - DEBUG: Disabled in production
  - INFO: Business events, API calls
  - WARN: Recoverable errors
  - ERROR: Exceptions, failed operations
  - CRITICAL: System failures

Structured Logging Format (JSON):
{
  "timestamp": "2025-02-15T10:30:00Z",
  "level": "INFO",
  "service": "listing-service",
  "trace_id": "abc123",  // For correlation
  "user_id": "user-789",
  "action": "create_listing",
  "message": "Listing created successfully",
  "metadata": {
    "listing_id": "listing-xyz",
    "residue_type": "rice_straw"
  }
}

Log Analysis:
  - CloudWatch Insights queries for troubleshooting
  - Anomaly detection for error spikes
  - Alerts for ERROR/CRITICAL logs
  - Retention: 90 days (hot), 1 year (S3 cold)
```

### 9.3 Distributed Tracing

```yaml
Tool: AWS X-Ray

Instrumentation:
  - API Gateway: Native integration
  - Lambda: X-Ray SDK
  - ECS: X-Ray daemon sidecar
  - RDS: Trace SQL queries (slow query log)

Trace Details:
  - Request path: Client → API Gateway → Lambda → RDS
  - Latency breakdown per service
  - Error identification
  - Service map visualization

Sampling:
  - Production: 5% of requests
  - Staging: 100% of requests
  - High-value transactions: 100% always
```

### 9.4 Alerting

```yaml
CloudWatch Alarms:

Critical Alarms (PagerDuty):
  - API Gateway 5xx > 1% (5 min period)
  - RDS CPU > 90% (5 min)
  - Lambda concurrent executions > 80% of limit
  - Payment failures > 5% (10 min)
  - Security: Unauthorized access attempts > 100/min

Warning Alarms (Slack):
  - API latency p95 > 500ms (15 min)
  - ECS CPU > 70% (10 min)
  - DynamoDB throttled requests > 10 (5 min)
  - Disk space > 80% (RDS)

Business Alarms (Email):
  - Daily listing creation drops > 30%
  - Transaction completion rate < 80%
  - User churn rate increases
  - Grading accuracy < 85%

Alarm Actions:
  - SNS topics → PagerDuty, Slack, Email
  - Auto-scaling triggers
  - Lambda function execution (remediation)
```

### 9.5 Performance Monitoring

```yaml
APM Tool: AWS X-Ray + CloudWatch ServiceLens

Key Metrics:
  - Apdex score (Application Performance Index)
  - Error rate (target: <1%)
  - Throughput (requests/second)
  - Response time: p50, p95, p99
  - Saturation (resource utilization)

Real User Monitoring (RUM):
  - Mobile app: Firebase Performance Monitoring
  - Web app: CloudWatch RUM
  - Metrics:
      - App launch time
      - Screen load time
      - Network request duration
      - Crash rate
      - Device/OS distribution

Synthetic Monitoring:
  - Tool: CloudWatch Synthetics (Canaries)
  - Tests:
      - API health checks (every 5 min)
      - Critical user journeys (every 15 min):
          1. User registration
          2. Listing creation
          3. Transaction flow
      - Uptime monitoring from multiple regions
```

---

## 10. Disaster Recovery

### 10.1 Backup Strategy

```yaml
RDS Backups:
  Automated Backups:
    - Frequency: Daily at 03:00 UTC
    - Retention: 30 days
    - Backup window: 03:00-04:00 UTC
    - Incremental (transaction logs every 5 min)
  
  Manual Snapshots:
    - Before major deployments
    - Before schema changes
    - Retention: Indefinite (tagged)
  
  Cross-Region Backup:
    - Daily snapshots replicated to ap-southeast-1
    - Automated via Lambda + RDS snapshot copy

DynamoDB Backups:
  Point-in-Time Recovery (PITR):
    - Enabled for all production tables
    - Restore to any second in last 35 days
  
  On-Demand Backups:
    - Weekly full table backups
    - Stored in S3 (DynamoDB exports)

S3 Backups:
  Versioning: Enabled on critical buckets
  Replication:
    - Cross-Region Replication (CRR) to ap-southeast-1
    - Replicate: All objects, delete markers
  
  Lifecycle Policies:
    - After 30 days → Glacier Instant Retrieval
    - After 90 days → Glacier Deep Archive
    - Compliance hold: 7 years for transaction data

Application Backups:
  - Container images: Retained in ECR (last 10 versions)
  - Lambda versions: Keep all versions (auto-prune after 90 days)
  - Configuration: Versioned in Git + S3
```

### 10.2 Disaster Recovery Plan

```yaml
DR Strategy: Pilot Light (cost-optimized)

Primary Region: ap-south-1 (Mumbai)
DR Region: ap-southeast-1 (Singapore)

Pilot Light Configuration:
  DR Region (Standby):
    - RDS: Read replica (running)
    - S3: Cross-region replication (active)
    - DynamoDB: Global tables (optional, high cost)
    - ECS: Task definitions ready, 0 tasks running
    - Lambda: Functions deployed, 0 invocations
    - Route53: Health checks configured

Failover Process:
  1. Trigger: Primary region outage detected (Route53 health check)
  
  2. Automated Actions (10 minutes):
     - Promote RDS read replica to primary (5 min)
     - Update Route53 DNS to point to DR region (TTL: 60s)
     - Scale up ECS services (0 → 5 tasks)
     - Warm up Lambda (pre-invoke concurrency)
  
  3. Manual Actions (30 minutes):
     - Verify data integrity
     - Update external integrations (payment gateway)
     - Communicate to users via status page
  
  4. Full Recovery:
     - RTO (Recovery Time Objective): 1 hour
     - RPO (Recovery Point Objective): 15 minutes

Failback Process:
  1. Primary region restored and tested
  2. Sync data from DR to primary (RDS, S3)
  3. Switch Route53 back to primary
  4. Scale down DR region to pilot light
  5. Post-mortem analysis

DR Testing:
  - Frequency: Quarterly
  - Scope: Full failover simulation
  - Duration: 4 hours
  - Success Criteria:
      - RTO < 1 hour
      - RPO < 15 minutes
      - No data loss
      - All critical APIs functional
```

### 10.3 High Availability Architecture

```yaml
Multi-AZ Deployment:
  - API Gateway: Multi-region (inherent)
  - Lambda: Runs across multiple AZs (AWS managed)
  - ECS Fargate: Tasks distributed across 2 AZs minimum
  - RDS: Multi-AZ with synchronous replication
  - ElastiCache: Redis cluster with replicas in 2 AZs
  - S3: 99.999999999% durability (built-in)

Load Balancing:
  Application Load Balancer:
    - Cross-zone load balancing: Enabled
    - Health checks: /health endpoint (30s interval)
    - Deregistration delay: 30 seconds
    - Sticky sessions: Disabled (stateless apps)
  
  Route53:
    - Latency-based routing (multi-region)
    - Health checks on ALB endpoints
    - Failover to DR region on primary failure

Auto-Scaling:
  ECS Services:
    - Target tracking: CPU 70%, Memory 80%
    - Step scaling: 
        - +2 tasks if CPU > 80% for 2 min
        - +5 tasks if CPU > 90% for 1 min
    - Scale-in: Gradual (1 task every 5 min)
  
  Lambda:
    - Provisioned concurrency: 10 (for critical functions)
    - Reserved concurrency: 100
    - Burst limit: 3000

Database High Availability:
  RDS:
    - Multi-AZ: Automatic failover (<1 min)
    - Read replicas: 2 (for scaling reads)
    - Connection pooling: RDS Proxy (99.99% SLA)
  
  DynamoDB:
    - Global tables (if multi-region needed)
    - On-demand capacity (auto-scales)
```

---

## 11. Cost Optimization

### 11.1 Cost Breakdown (Estimated)

```yaml
Monthly AWS Costs (Production, at scale):

Compute:
  - ECS Fargate: $800 (10 services, avg 5 tasks each)
  - Lambda: $400 (5M invocations, 500ms avg)
  - SageMaker: $1200 (2 endpoints, ml.c5.xlarge)
  - Total: $2,400

Storage:
  - S3: $150 (5 TB images + data lake)
  - RDS: $600 (db.r6g.xlarge Multi-AZ)
  - DynamoDB: $200 (On-Demand, 10M reads, 5M writes)
  - ElastiCache: $250 (cache.r6g.large)
  - Total: $1,200

Networking:
  - Data Transfer Out: $300 (3 TB/month)
  - NAT Gateway: $100
  - CloudFront: $150
  - Total: $550

AI/ML:
  - SageMaker Training: $200 (weekly retraining)
  - Rekognition: $50
  - Total: $250

Monitoring & Security:
  - CloudWatch: $100
  - X-Ray: $50
  - WAF: $100
  - Total: $250

**TOTAL: ~$4,650/month** (with 10,000 active users)

Cost Optimization Strategies:
  - Spot instances for batch jobs (50% savings)
  - S3 Intelligent-Tiering (30% storage savings)
  - Reserved instances for predictable workloads (40% compute savings)
  - Lambda ARM Graviton2 (20% savings)
  - Savings Plans commitment (15-20% overall)
```

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-02-15 | Agri-Chakra Team | Initial system design specification |

---

**Note**: This design document is a living artifact and will evolve as we iterate on the architecture, incorporate learnings from production, and adapt to changing requirements.
