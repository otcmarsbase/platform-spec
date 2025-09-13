# MB Community Deals Platform - Technical Specification

## Executive Summary

MB Community Deals is a sophisticated Web3 investment platform built on Next.js 14 that enables issuers to create tokenized investment opportunities and investors to participate in deals through a secure, compliance-focused infrastructure. The platform supports multi-tenant deployments, blockchain integration across multiple chains, and comprehensive KYC/AML compliance.

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Technology Stack](#technology-stack)
3. [Database Schema](#database-schema)
4. [Core Modules](#core-modules)
5. [API Architecture](#api-architecture)
6. [Frontend Architecture](#frontend-architecture)
7. [Blockchain Integration](#blockchain-integration)
8. [Security & Compliance](#security--compliance)
9. [Infrastructure Requirements](#infrastructure-requirements)
10. [Deployment Guide](#deployment-guide)

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend Layer                          │
│  Next.js 14 App Router │ React 18 │ TailwindCSS │ TypeScript   │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                              │
│    RESTful APIs │ Next.js API Routes │ Authentication           │
├─────────────────────────────────────────────────────────────────┤
│                      Business Logic Layer                       │
│  Deal Management │ Investment Groups │ User Management │ KYC    │
├─────────────────────────────────────────────────────────────────┤
│                       Data Access Layer                         │
│     PostgreSQL │ Repository Pattern │ Query Builders            │
├─────────────────────────────────────────────────────────────────┤
│                    Blockchain Integration                       │
│  Thirdweb SDK │ Ethers.js v6 │ Multi-chain Support             │
├─────────────────────────────────────────────────────────────────┤
│                    External Services                            │
│  IPFS │ Sumsub KYC │ ShuftiPro │ Email (Resend) │ Telegram     │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-Tenant Architecture

The platform supports multiple tenants with isolated data and customizable features:
- Tenant-specific database isolation
- Custom branding and theming
- Configurable features and modules
- Independent KYC provider settings
- Custom domain support

## Multi-Tenant Architecture Deep Dive

### Overview

MB Community Deals implements a sophisticated multi-tenant architecture that enables white-label deployments while maintaining complete data isolation, security, and customization capabilities. Each tenant operates as an independent platform instance with its own users, deals, configurations, and branding.

### Tenant Isolation Model

#### 1. Data Isolation Strategy

**Row-Level Security (RLS)**
- Every database table includes a `tenant_id` column
- PostgreSQL Row-Level Security policies enforce isolation
- Automatic tenant context injection in all queries
- Cross-tenant data access is impossible at the database level

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                     │
├─────────────────────────────────────────────────────────┤
│                 Tenant Context Middleware                │
│         (Extracts tenant from domain/subdomain)         │
├─────────────────────────────────────────────────────────┤
│                  Tenant-Aware ORM Layer                  │
│            (Automatically filters by tenant_id)          │
├─────────────────────────────────────────────────────────┤
│                    PostgreSQL Database                   │
│         (Row-Level Security + tenant_id column)         │
└─────────────────────────────────────────────────────────┘
```

#### 2. Tenant Identification Methods

**Domain-Based Routing**
- Primary domain: `tenant1.mbcommunitydeals.com`
- Custom domain: `invest.clientcompany.com`
- Domain alias support for multiple URLs per tenant

**Tenant Resolution Flow**
1. Request arrives at platform
2. Domain/subdomain extracted from request
3. Tenant lookup in `tenants` and `domain_aliases` tables
4. Tenant context established for entire request lifecycle
5. All subsequent queries filtered by tenant_id

### Tenant Configuration Architecture

#### 1. Core Tenant Settings

**Database Schema**
```
tenants table:
- id: UUID (primary key)
- name: Platform display name
- slug: URL-safe identifier
- domain: Primary domain
- settings: JSONB configuration
- features: JSONB feature flags
- theme_settings: JSONB branding
- kyc_provider: Selected KYC service
- blockchain_settings: Chain configurations
- created_at: Timestamp
```

#### 2. Configurable Features Per Tenant

**Feature Categories**
- **Investment Features**
  - Escrow system (on/off)
  - P2P trading (on/off)
  - Secondary market (on/off)
  - Investment groups (on/off)
  - Asset tokenization (on/off)

- **Compliance Features**
  - KYC provider selection (Sumsub/ShuftiPro/Custom)
  - Verification levels required
  - Geographic restrictions
  - Accreditation requirements
  - Investment limits

- **Blockchain Features**
  - Supported chains (Ethereum/Polygon/Base/etc.)
  - Smart contract addresses
  - Gas sponsorship
  - Multi-token support

- **UI/UX Features**
  - Custom navigation menus
  - Homepage layout
  - Dashboard widgets
  - Social features (on/off)
  - Language support

#### 3. Branding & Theming System

**Customizable Elements**
```json
{
  "theme_settings": {
    "colors": {
      "primary": "#007AFF",
      "secondary": "#5856D6",
      "accent": "#FF3B30",
      "background": "#FFFFFF",
      "text": "#000000"
    },
    "typography": {
      "fontFamily": "Inter",
      "headingFont": "Poppins",
      "fontSize": {
        "base": "16px",
        "scale": 1.25
      }
    },
    "logo": {
      "light": "https://cdn.example.com/logo-light.svg",
      "dark": "https://cdn.example.com/logo-dark.svg",
      "favicon": "https://cdn.example.com/favicon.ico"
    },
    "layout": {
      "style": "modern", // modern/classic/minimal
      "sidebarPosition": "left",
      "headerStyle": "floating"
    }
  }
}
```

### Tenant Lifecycle Management

#### 1. Tenant Onboarding Process

**Automated Provisioning Flow**
1. **Registration**
   - Company information collection
   - Admin user creation
   - Subscription plan selection

2. **Configuration**
   - Domain setup (subdomain or custom)
   - SSL certificate provisioning
   - Initial branding configuration
   - Feature selection

3. **Data Setup**
   - Database schema initialization
   - Default data seeding
   - Smart contract deployment (if needed)
   - Test deal creation (optional)

4. **Integration**
   - KYC provider API keys
   - Email service configuration
   - Blockchain RPC endpoints
   - Payment gateway setup

5. **Launch**
   - Health check validation
   - Admin training
   - Go-live activation

#### 2. Tenant Management Interface

**Super Admin Dashboard**
- Create new tenants
- Monitor tenant health
- Usage analytics per tenant
- Billing and subscription management
- Feature flag control
- Emergency access controls

**Tenant Admin Dashboard**
- Branding configuration
- User management
- Feature toggles
- Integration settings
- Analytics and reporting
- Billing information

### Technical Implementation Details

#### 1. Middleware Architecture

**Tenant Context Middleware**
```typescript
// Establishes tenant context for every request
export async function tenantMiddleware(request: Request) {
  const hostname = request.headers.get('host');
  const tenant = await resolveTenant(hostname);
  
  if (!tenant) {
    return new Response('Tenant not found', { status: 404 });
  }
  
  // Inject tenant context
  request.tenantId = tenant.id;
  request.tenantConfig = tenant.settings;
  
  return NextResponse.next();
}
```

#### 2. Database Query Interception

**Automatic Tenant Filtering**
```typescript
// All queries automatically filtered by tenant
export class TenantAwareRepository {
  async find(conditions: any) {
    const tenantId = getCurrentTenantId();
    return await pool.query(
      'SELECT * FROM table WHERE tenant_id = $1 AND ...',
      [tenantId, ...params]
    );
  }
}
```

#### 3. Smart Contract Multi-Tenancy

**Contract Deployment Strategy**
- **Shared Contracts**: Platform-wide contracts (service fees)
- **Tenant Contracts**: Isolated per tenant (token sales, pools)
- **Factory Pattern**: Dynamic contract creation per deal

```
Platform Service Contract (Shared)
    ├── Tenant A Factory
    │   ├── Deal 1 TokenSale
    │   ├── Deal 2 TokenSale
    │   └── Escrow Contracts
    └── Tenant B Factory
        ├── Deal 1 TokenSale
        ├── Deal 2 TokenSale
        └── Escrow Contracts
```

### Tenant Data Management

#### 1. Data Segregation

**Complete Isolation**
- Users cannot access other tenant data
- Investments isolated per tenant
- Documents and media separated
- Analytics scoped to tenant

**Shared Resources** (Platform Level)
- Smart contract ABIs
- System configurations
- Platform updates
- Security patches

#### 2. Backup & Recovery

**Tenant-Specific Backups**
- Individual tenant data export
- Point-in-time recovery per tenant
- Selective restoration
- Data portability for tenant migration

### Performance Optimization

#### 1. Caching Strategy

**Multi-Level Cache**
```
CDN (CloudFlare) → Tenant-specific cache keys
    ↓
Application Cache (Redis) → Isolated by tenant_id
    ↓
Database Query Cache → Tenant-filtered results
```

#### 2. Resource Allocation

**Fair Resource Sharing**
- Connection pool limits per tenant
- API rate limiting per tenant
- Storage quotas
- Bandwidth allocation

### Security Considerations

#### 1. Tenant Isolation Security

**Access Control**
- Tenant-specific JWT claims
- Role-based permissions per tenant
- API key scoping
- Webhook URL validation

**Audit Trail**
- Tenant-specific audit logs
- Cross-tenant access attempts logged
- Admin action tracking
- Compliance reporting per tenant

#### 2. Data Privacy

**GDPR Compliance**
- Tenant-specific data processing agreements
- Individual data export capabilities
- Right to be forgotten per tenant
- Data residency options

### Scaling Multi-Tenant Platform

#### Horizontal Scaling

**Tenant Sharding Options**
- Database sharding by tenant_id
- Geographical distribution
- Read replica per major tenant
- Dedicated resources for enterprise tenants


### Migration & Portability

#### 1. Tenant Migration

**Export Capabilities**
- Complete data export in standard formats
- Smart contract state export
- User credentials migration
- Transaction history preservation

#### 2. White-Label Deployment

**Standalone Deployment Option**
- Extract single tenant to independent instance
- Maintain data integrity
- Preserve transaction history
- Smart contract migration tools

### Monitoring & Analytics

#### 1. Tenant Metrics

**Key Performance Indicators**
- Active users per tenant
- Transaction volume
- Storage usage
- API usage
- Feature adoption rates

#### 2. Health Monitoring

**Per-Tenant Health Checks**
- Database connection health
- API response times
- Smart contract status
- Integration health
- Error rates

### Billing & Subscription Management

#### 1. Usage-Based Billing

**Metered Resources**
- Number of active users
- Transaction volume
- Storage consumption
- API calls
- Smart contract deployments

#### 2. Subscription Models

**Flexible Plans**
- Monthly/Annual billing
- Feature-based pricing
- User-based pricing
- Transaction fee models
- Custom enterprise agreements

## Technology Stack

### Core Technologies

| Category | Technology | Version | Purpose |
|----------|-----------|---------|---------|
| **Runtime** | Node.js | 18+ | Server runtime |
| **Framework** | Next.js | 14.1.0 | Full-stack React framework |
| **Language** | TypeScript | 5.3.3 | Type-safe JavaScript |
| **Database** | PostgreSQL | 14+ | Primary data store |
| **UI Framework** | React | 18.2.0 | Component library |
| **Styling** | TailwindCSS | 3.4.1 | Utility-first CSS |
| **Blockchain** | Thirdweb | 5.105.33 | Web3 SDK |
| **Blockchain** | Ethers.js | 6.13.5 | Ethereum library |

### Key Dependencies

#### Authentication & Security
- `jsonwebtoken`: JWT token management
- `bcrypt`: Password hashing
- `jose`: JWT validation
- `helmet`: Security headers

#### Blockchain & Web3
- `thirdweb`: Complete Web3 SDK for smart contract interaction
- `ethers`: Ethereum wallet and contract interaction
- `wagmi`: React hooks for Ethereum

#### Data Management
- `pg`: PostgreSQL client
- `swr`: Data fetching and caching
- `react-hook-form`: Form management
- `zod`: Schema validation

#### UI Components
- `@radix-ui/*`: Accessible component primitives
- `lucide-react`: Icon library
- `framer-motion`: Animation library
- `recharts`: Data visualization

#### External Services
- `@sumsub/websdk-react`: KYC verification
- `resend`: Email service
- `openai`: AI integration
- `axios`: HTTP client

## Database Schema

### Core Database Tables

The platform uses PostgreSQL with the following table structure organized by functional domain:

#### Users & Authentication Tables
- **users** - Core user accounts with authentication credentials
- **profiles** - Extended user information including KYC status and verification levels
- **user_verification_levels** - Tracks user verification progression
- **verification_codes** - Temporary codes for email/phone verification
- **sessions** - Active user sessions management

#### Multi-Tenancy Tables
- **tenants** - Platform instances with custom configurations
- **tenant_user_roles** - User roles and permissions per tenant
- **tenant_features** - Enabled features per tenant
- **tenant_metadata** - Additional tenant configuration
- **tenant_settings** - Tenant-specific settings
- **domain_aliases** - Custom domain mapping for tenants

#### Deal Management Tables
- **deals** - Investment opportunities and their parameters
- **investments** - User investments in deals
- **deal_documents** - Documents associated with deals
- **deal_media** - Media files (images, videos) for deals
- **deal_updates** - Updates and announcements for deals
- **deal_faqs** - Frequently asked questions per deal
- **deal_key_terms** - Key investment terms
- **deal_required_documents** - Required documents from investors
- **deal_reviews** - User reviews and ratings
- **deal_comments** - User comments and discussions
- **deal_groups** - Grouped deals for campaigns

#### Investment Groups Tables
- **investment_groups** - Exclusive investment communities
- **group_memberships** - Group member management
- **group_deals** - Deals exclusive to groups
- **group_applications** - Membership applications

#### Escrow System Tables
- **escrow_investments** - KYC-pending investments in escrow
- **escrow_transactions** - Escrow transaction history
- **escrow_refunds** - Refund tracking

#### Asset Tokenization Tables
- **assets** - Tokenized real-world assets
- **asset_applications** - Tokenization requests
- **asset_updates** - Asset value and status updates
- **asset_transactions** - Asset trading history

#### Issuer Management Tables
- **issuers** - Registered deal issuers
- **issuer_profiles** - Detailed issuer information
- **issuer_documents** - Issuer verification documents
- **legal_entities** - Legal entity information

#### KYC/Compliance Tables
- **kyc_applications** - KYC application records
- **kyc_documents** - Uploaded KYC documents
- **compliance_checks** - Compliance verification records
- **suitability_assessments** - Investor suitability checks

#### P2P Trading Tables
- **p2p_trades** - Peer-to-peer token trades
- **p2p_offers** - Trade offers
- **p2p_transactions** - P2P transaction history
- **pro_trade_orders** - Professional trading orders

#### Notification System Tables
- **notifications** - User notifications
- **notification_preferences** - User notification settings
- **email_logs** - Email delivery tracking
- **telegram_pins** - Telegram integration

#### Activity & Audit Tables
- **activity_logs** - Comprehensive activity tracking
- **audit_logs** - Security and compliance audit trail
- **admin_audit_summary** - Aggregated admin activity

#### Broker & Transfer Agent Tables
- **brokers** - Registered brokers
- **broker_settings** - Broker configurations
- **transfer_agents** - Transfer agent records
- **transfer_agent_settings** - Transfer agent configurations

#### Referral System Tables
- **referrers** - Referral partners
- **referral_links** - Tracking links
- **referral_transactions** - Referral rewards

#### Lending Protocol Tables
- **lending_pools** - DeFi lending pools
- **user_lending_positions** - User lending positions
- **lending_transactions** - Lending transaction history

#### Staking System Tables
- **staking_pools** - Token staking pools
- **user_stakes** - User staking positions
- **staking_rewards** - Reward distributions

#### Social Features Tables
- **user_feed_posts** - Social feed posts
- **post_comments** - Comments on posts
- **post_engagements** - Likes and reactions
- **user_follows** - Social following relationships

#### Document Management Tables
- **documents** - General document storage
- **document_versions** - Document version history
- **document_access_logs** - Document access tracking

#### Newsletter & Marketing Tables
- **newsletter_subscribers** - Email subscribers
- **newsletter_campaigns** - Email campaigns
- **click_tracking** - Link click analytics

### Database Indexes

Key performance indexes:
- User lookups: `idx_users_email`, `idx_users_username`
- Deal queries: `idx_deals_tenant_status`, `idx_deals_issuer`
- Investment tracking: `idx_investments_deal_investor`
- Group memberships: `idx_group_members_status`
- Activity logs: `idx_activity_logs_entity_created`

## Core Modules

### 1. Authentication & Authorization Module

**Location**: `/lib/auth/`

**Components**:
- `middleware.ts`: Core authentication middleware
- `auth-helpers.ts`: JWT validation and user verification
- `issuer-middleware.ts`: Issuer-specific authorization
- `broker-middleware.ts`: Broker role authorization
- `transfer-agent-middleware.ts`: Transfer agent permissions
- `audit-middleware.ts`: Audit logging integration

**Features**:
- JWT-based authentication
- Role-based access control (RBAC)
- Multi-tenant user isolation
- Session management
- Password hashing with bcrypt
- Custom JWT providers (BeBelong, Custom)

### 2. Deal Management Module

**Location**: `/lib/db/repositories/deal-repository.ts`

**Features**:
- Deal CRUD operations
- Advanced filtering and search
- Status workflow management
- Document and media management
- Investment tracking
- Pool progress monitoring
- Multi-chain deployment

**Key Methods**:
```typescript
- getDeals(filters): Retrieve filtered deals
- createDeal(data): Create new investment opportunity
- updateDeal(id, data): Update deal information
- deployContracts(id): Deploy smart contracts
- trackInvestment(dealId, investment): Record investment
```

### 3. Investment Groups Module

**Location**: `/app/api/groups/`, `/app/api/issuer/groups/`

**Features**:
- One group per issuer limitation
- Member-exclusive deal access
- Application workflow
- Public/private group visibility
- Custom requirements and focus areas
- Visual branding with IPFS

**Access Control**:
- Public groups visible in discovery
- Group deals only visible to approved members
- Application review by group leaders

### 4. Escrow Investment System

**Location**: `/constants/contracts-escrow.ts`

**Features**:
- Individual escrow contracts per investment
- 30-day maximum duration
- 14-day admin action deadline
- Multiple refund mechanisms
- AML-compliant transaction flow
- Time-based protection

**Smart Contract Integration**:
```typescript
- EscrowFactory: Creates individual escrows
- Escrow: Holds funds during KYC
- TokenSale: Receives approved funds
- Refund mechanisms: Rejection, timeout, manual
```

### 5. KYC/AML Compliance Module

**Location**: `/lib/kycService.ts`, `/lib/kycWebhook.ts`

**Providers**:
- Sumsub: Primary KYC provider
- ShuftiPro: Alternative provider
- Custom verification levels

**Verification Levels**:
1. Basic: Email verification
2. Standard: Identity verification
3. Enhanced: Full KYC/AML
4. Accredited: Investor accreditation

### 6. Blockchain Integration Module

**Location**: `/lib/client.ts`, `/constants/contracts.ts`

**Supported Chains**:
- Ethereum Mainnet
- Polygon
- Base
- Arbitrum
- Optimism

**Features**:
- Multi-chain smart contract deployment
- Token sale contract management
- Pool contract integration
- Transaction monitoring
- Gas optimization
- Contract verification

### 7. Document Management Module

**Location**: `/lib/document-storage.ts`, `/lib/ipfs.ts`

**Features**:
- IPFS integration for decentralized storage
- Document versioning
- Access control
- Encryption support
- Multiple document types
- Automatic thumbnails

### 8. Notification System

**Location**: `/lib/db/notifications.ts`

**Channels**:
- Email (Resend API)
- In-app notifications
- Telegram integration
- Webhook support

**Event Types**:
- Investment confirmations
- KYC status updates
- Deal status changes
- Group applications
- System announcements

## API Architecture

### RESTful API Structure

```
/api/
├── auth/
│   └── [...nextauth]/         # Authentication endpoints
├── admin/
│   ├── deals/                 # Deal administration
│   ├── users/                 # User management
│   ├── applications/          # Application review
│   ├── assets/                # Asset management
│   └── stats/                 # Platform analytics
├── deals/
│   ├── [id]/
│   │   ├── invest/           # Investment endpoint
│   │   ├── applications/     # Deal applications
│   │   └── documents/        # Deal documents
│   └── featured/             # Featured deals
├── groups/
│   ├── [id]/
│   │   ├── apply/            # Group application
│   │   ├── dashboard/        # Member dashboard
│   │   └── deals/            # Group deals
│   └── my-groups/            # User's groups
├── issuer/
│   ├── deals/                # Issuer deal management
│   ├── groups/               # Group management
│   └── applications/         # Application review
├── investor/
│   ├── portfolio/            # Investment portfolio
│   ├── kyc/                  # KYC management
│   └── documents/            # Document upload
├── profile/
│   ├── settings/             # User settings
│   ├── wallet/               # Wallet management
│   └── verification/         # Verification status
└── webhooks/
    ├── kyc/                  # KYC provider webhooks
    └── blockchain/           # Blockchain events
```

### API Authentication Flow

1. **Login Request**: User provides credentials
2. **Validation**: Verify against database
3. **JWT Generation**: Create signed token
4. **Response**: Return token to client
5. **Subsequent Requests**: Include token in headers
6. **Middleware Validation**: Verify token on each request

### Rate Limiting & Throttling

- Public endpoints: 100 requests/minute
- Authenticated endpoints: 500 requests/minute
- Admin endpoints: 1000 requests/minute
- Webhook endpoints: No limit

## Frontend Architecture

### Component Structure

```
/components/
├── ui/                    # Reusable UI components
│   ├── button.tsx
│   ├── card.tsx
│   ├── dialog.tsx
│   └── ...
├── deals/                 # Deal-specific components
│   ├── deal-card.tsx
│   ├── deal-form.tsx
│   └── investment-modal.tsx
├── groups/                # Investment group components
│   ├── group-card.tsx
│   ├── group-form.tsx
│   └── member-list.tsx
├── admin/                 # Admin interface components
│   ├── application-detail.tsx
│   ├── deal-review.tsx
│   └── user-management.tsx
├── wallet/                # Web3 wallet components
│   ├── connect-button.tsx
│   └── transaction-list.tsx
└── layout/                # Layout components
    ├── header.tsx
    ├── sidebar.tsx
    └── footer.tsx
```

### State Management

- **Server State**: SWR for data fetching and caching
- **Client State**: React hooks and context
- **Form State**: React Hook Form
- **Web3 State**: Thirdweb hooks

### Routing Structure

Using Next.js 14 App Router:
```
/app/
├── (auth)/               # Authentication pages
│   ├── login/
│   └── signup/
├── (dashboard)/          # Main application
│   ├── deals/
│   ├── groups/
│   ├── portfolio/
│   └── profile/
├── admin/                # Admin panel
│   ├── dashboard/
│   ├── deals/
│   └── users/
└── api/                  # API routes
```

### Design System

- **Framework**: TailwindCSS with custom configuration
- **Components**: Radix UI primitives
- **Icons**: Lucide React
- **Animations**: Framer Motion
- **Charts**: Recharts
- **Theme**: Dark/Light mode support

## Blockchain Integration

### Smart Contract Architecture

#### 1. TokenSale Contract
- Manages token sales
- Handles investments
- Distributes tokens
- Tracks allocations

#### 2. Pool Contract
- Manages investment pools
- Tracks contributions
- Handles distributions
- Manages refunds

#### 3. Escrow Contract System
- EscrowFactory: Creates individual escrows
- Escrow: Holds funds during KYC
- Time-based expiration
- Multiple refund paths

#### 4. Service Contract
- Platform fee management
- Revenue distribution
- Admin functions
- Emergency controls

### Contract Deployment Flow

1. **Preparation**: Validate deal parameters
2. **Token Creation**: Deploy or connect token contract
3. **TokenSale Deployment**: Deploy sale contract
4. **Pool Creation**: Deploy pool if needed
5. **Configuration**: Set parameters and permissions
6. **Verification**: Verify on block explorer
7. **Activation**: Enable investments

### Transaction Management

- **Gas Optimization**: Batch operations when possible
- **Error Handling**: Comprehensive error messages
- **Retry Logic**: Automatic retry for failed transactions
- **Monitoring**: Real-time transaction status
- **Receipts**: Store transaction hashes

## Security & Compliance

### Security Measures

#### 1. Application Security
- Input validation and sanitization
- SQL injection prevention (parameterized queries)
- XSS protection
- CSRF tokens
- Rate limiting
- Security headers (Helmet.js)

#### 2. Authentication Security
- Bcrypt password hashing
- JWT with expiration
- Refresh token rotation
- Multi-factor authentication support
- Session management

#### 3. Data Security
- Encryption at rest
- TLS/SSL for data in transit
- Sensitive data masking
- Audit logging
- GDPR compliance

#### 4. Smart Contract Security
- Audited contracts
- Reentrancy guards
- Access controls
- Emergency pause
- Upgrade mechanisms

### Compliance Features

#### 1. KYC/AML Integration
- Multiple provider support
- Tiered verification
- Document verification
- Sanctions screening
- PEP checks

#### 2. Regulatory Compliance
- Reg D/S support
- Accredited investor verification
- Investment limits
- Geographic restrictions
- Reporting tools

#### 3. Audit Trail
- Complete activity logging
- User action tracking
- Transaction history
- Document versioning
- Compliance reporting

## Infrastructure Requirements

### Minimum Requirements

#### Production Environment
- **CPU**: 4 vCPUs minimum
- **RAM**: 8GB minimum
- **Storage**: 100GB SSD
- **Network**: 1Gbps connection
- **Database**: PostgreSQL 14+
- **Node.js**: 18+ LTS

#### Development Environment
- **CPU**: 2 vCPUs
- **RAM**: 4GB
- **Storage**: 50GB
- **Node.js**: 18+
- **Database**: PostgreSQL 14+

### Recommended Stack

#### Cloud Providers
- **AWS**: EC2, RDS, S3, CloudFront
- **Google Cloud**: Compute Engine, Cloud SQL
- **Azure**: Virtual Machines, Database
- **Vercel**: Frontend hosting (recommended)

#### Supporting Services
- **CDN**: CloudFlare or CloudFront
- **Monitoring**: DataDog or New Relic
- **Logging**: LogRocket or Sentry
- **Analytics**: Google Analytics or Mixpanel
- **Email**: Resend or SendGrid

### Scaling Considerations

#### Horizontal Scaling
- Load balancer configuration
- Database read replicas
- Redis caching layer
- CDN for static assets
- Microservices architecture

#### Vertical Scaling
- Increase server resources
- Database optimization
- Query performance tuning
- Index optimization
- Connection pooling

## Deployment Guide

### Prerequisites

1. **Environment Setup**
```bash
# Install Node.js 18+
# Install PostgreSQL 14+
# Install Git
```

2. **Clone Repository**
```bash
git clone [repository-url]
cd cdao-community-deals
```

3. **Install Dependencies**
```bash
npm install
```

4. **Environment Configuration**
```bash
cp .env.example .env.local
# Edit .env.local with your configuration
```

### Database Setup

1. **Create Database**
```sql
CREATE DATABASE cdao_community_deals;
```

2. **Run Migrations**
```bash
npm run migrate
```

3. **Seed Data** (Optional)
```bash
npm run seed
```

### Build & Deploy

#### Production Build
```bash
npm run build
npm run start
```

#### Docker Deployment
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

#### Vercel Deployment
```bash
npm i -g vercel
vercel --prod
```

### Post-Deployment

1. **Verify Services**
   - Database connectivity
   - External API connections
   - Blockchain RPC endpoints
   - Email service
   - KYC provider integration

2. **Security Checklist**
   - Enable HTTPS
   - Configure firewall
   - Set up monitoring
   - Enable backups
   - Configure alerts

3. **Performance Optimization**
   - Enable caching
   - Optimize images
   - Minify assets
   - Enable compression
   - Configure CDN

## Monitoring & Maintenance

### Health Checks

```typescript
// API health check endpoint
GET /api/health

Response:
{
  "status": "healthy",
  "database": "connected",
  "blockchain": "connected",
  "services": {
    "kyc": "operational",
    "email": "operational",
    "ipfs": "operational"
  }
}
```

### Backup Strategy

- **Database**: Daily automated backups
- **Files**: IPFS pinning + S3 backup
- **Configuration**: Version controlled
- **Retention**: 30 days minimum

### Update Process

1. **Staging Deployment**: Test updates
2. **Database Migration**: Run if needed
3. **Blue-Green Deployment**: Zero downtime
4. **Rollback Plan**: Previous version ready
5. **Verification**: Automated tests

## API Documentation

### Authentication Endpoints

#### POST /api/auth/login
```json
Request:
{
  "email": "user@example.com",
  "password": "securepassword"
}

Response:
{
  "token": "jwt-token",
  "user": {
    "id": 1,
    "email": "user@example.com",
    "role": "investor"
  }
}
```

#### POST /api/auth/register
```json
Request:
{
  "email": "user@example.com",
  "password": "securepassword",
  "firstName": "John",
  "lastName": "Doe"
}

Response:
{
  "success": true,
  "message": "Registration successful"
}
```

### Deal Endpoints

#### GET /api/deals
```json
Query Parameters:
- status: active|pending|completed
- assetType: equity|debt|token
- search: string
- limit: number
- offset: number

Response:
{
  "deals": [...],
  "total": 100,
  "hasMore": true
}
```

#### POST /api/deals/[id]/invest
```json
Request:
{
  "amount": 10000,
  "paymentMethod": "crypto",
  "walletAddress": "0x..."
}

Response:
{
  "success": true,
  "investment": {
    "id": "uuid",
    "transactionHash": "0x..."
  }
}
```

## Troubleshooting Guide

### Common Issues

#### Database Connection Errors
```bash
# Check PostgreSQL service
sudo systemctl status postgresql

# Verify connection string
psql $DATABASE_URL
```

#### Build Failures
```bash
# Clear cache
rm -rf .next node_modules
npm install
npm run build
```

#### Blockchain Connection Issues
```bash
# Test RPC endpoint
curl -X POST $RPC_URL \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

### Performance Issues

1. **Slow Queries**: Check database indexes
2. **High Memory**: Review connection pools
3. **Slow Page Load**: Enable caching
4. **Transaction Failures**: Check gas prices

## Support & Resources

### Documentation
- Technical Documentation: This document
- API Reference: `/docs/api`
- Smart Contract Docs: `/docs/contracts`
- User Guide: `/docs/user-guide`

### Community
- GitHub Issues: Report bugs
- Discord: Community support
- Email: technical@marsbase.network

### License
PROPRIETARY SOFTWARE LICENSE

This software and associated documentation files (the "Software") are the proprietary property of MB Community Deals. All rights reserved.

The Software is licensed, not sold. This license grants the purchaser ("Licensee") the following rights:

1. **Single Installation**: The right to install and use one instance of the Software on premises owned or controlled by the Licensee
2. **Modifications**: The right to modify the Software for internal use only
3. **Internal Use**: The Software may only be used for Licensee's internal business purposes

**Restrictions**:
- No redistribution or resale of the Software
- No sublicensing to third parties
- No use of the Software for competing services
- Source code must remain confidential

For licensing inquiries, contact: licensing@marsbase.network

---

## Appendix A: Environment Variables

```env
# Core Configuration
NODE_ENV=production
DATABASE_URL=postgresql://user:pass@host:5432/db

# Blockchain
NEXT_PUBLIC_THIRDWEB_CLIENT_ID=
THIRDWEB_SECRET_KEY=
NEXT_PUBLIC_CHAIN_ID=137

# KYC Providers
SUMSUB_APP_TOKEN=
SUMSUB_SECRET_KEY=
SHUFTI_CLIENT_ID=
SHUFTI_SECRET_KEY=

# External Services
RESEND_API_KEY=
TELEGRAM_BOT_TOKEN=
IPFS_API_KEY=

# Contract Addresses
NEXT_PUBLIC_TOKEN_SALE_ADDRESS=
NEXT_PUBLIC_ESCROW_FACTORY_ADDRESS=
NEXT_PUBLIC_SERVICE_CONTRACT_ADDRESS=

# URLs
FRONTEND_URL=https://yourdomain.com
API_URL=https://api.yourdomain.com
```

*Document Version: 1.0.0*
*Last Updated: January 2025*
*Prepared for: On-Premise Deployment*
