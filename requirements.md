# Agri-Chakra: MVP Requirements (Ideation Phase)

## Problem Statement

**Agricultural waste crisis**: 500M+ tons of crop residues burned annually in India, causing severe air pollution. Farmers lack access to fair-priced buyers, while industries struggle to source quality biomass. Informal middlemen exploit both parties.

**Agri-Chakra Solution**: AI-powered marketplace connecting farmers directly with industrial buyers through automated waste grading, fair pricing, and digital transactions.

---

## MVP Scope (3-Month Timeline)

### Core Features

#### 1. User Management
- **Farmer Registration**: Phone-based signup with OTP verification
- **Buyer Registration**: Basic company details and GST verification
- **User Profiles**: Simple dashboard for tracking listings/transactions

- **User Profiles**: Simple dashboard for tracking listings/transactions

#### 2. AI-Powered Waste Grading
- **Image Upload**: Farmers capture residue images via mobile app
- **Automated Classification**: 
  - Residue type detection (rice straw, bagasse, wheat straw)
  - Quality grading (A/B/C based on visual analysis)
  - Quantity estimation from images
- **Instant Results**: Grade + confidence score in <5 seconds

#### 3. Smart Pricing
- **Price Recommendation**: AI suggests fair price based on quality, location, market trends
- **Dynamic Pricing**: Updates based on demand-supply data

#### 4. Marketplace
- **Listing Creation**: Farmers post graded residues with photos, quantity, price
- **Search & Discovery**: Buyers filter by residue type, quality, location, quantity
- **Basic Matching**: System suggests relevant listings to buyers

#### 5. Transaction Flow
- **Booking System**: Buyers reserve listings
- **Digital Contracts**: Auto-generated agreement templates
- **Payment Integration**: UPI/payment gateway for deposits and final settlement
- **Status Tracking**: Real-time updates (booked → delivered → completed)

#### 6. Chatbot Assistant
- **Farmer Support**: WhatsApp/in-app bot for listing help, price queries
- **Buyer Queries**: Answer questions about residue types, quality standards
- **Multi-language**: Hindi, English, Punjabi

---

## MVP Tech Stack (Serverless AWS)

### Core Services
- **Frontend**: React (Web) + React Native (Mobile)
- **Authentication**: Amazon Cognito (phone-based OTP)
- **API Layer**: Amazon API Gateway
- **Business Logic**: AWS Lambda (Node.js/Python)
- **Database**: Amazon DynamoDB (listings, users, transactions)
- **Storage**: Amazon S3 (images, documents)

### AI/ML Services
- **Image Analysis**: Amazon Rekognition (custom labels for grading)
- **Chatbot**: Amazon Bedrock (GenAI for conversational support)
- **Predictions**: Amazon SageMaker (price forecasting, yield estimation)

### Supporting Services
- **Notifications**: Amazon SNS (SMS/email)
- **Monitoring**: Amazon CloudWatch
- **Deployment**: AWS Amplify (frontend hosting)

---

## Success Metrics (MVP)

### Adoption
- 500 farmers onboarded (pilot region: Punjab/Haryana)
- 20 industrial buyers registered
- 1,000 tons biomass listed in first 3 months

### Technical
- 85%+ AI grading accuracy
- <5 sec grading time
- 90%+ app uptime

### Impact
- 100 tons residue traded (prevent stubble burning)
- ₹5L+ revenue for farmers
- 10% increase in farmer income vs. local rates

---

## Out of Scope (Post-MVP)
- Logistics integration
- Payment escrow
- Advanced aggregation algorithms
- Blockchain traceability
- Carbon credit integration
- Multi-region deployment

---

## Assumptions
- Farmers have smartphones with camera
- 3G+ internet connectivity in pilot regions
- Industrial buyers willing to pilot new procurement channel
- Government support/subsidies available for adoption
