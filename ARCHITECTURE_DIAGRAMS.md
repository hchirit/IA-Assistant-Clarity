# Clarity PPM AI Chatbot - Architecture & Technical Specifications

## Table of Contents

1. [System Architecture Overview](#system-architecture-overview)
2. [AWS Deployment Architecture](#aws-deployment-architecture)
3. [Authentication & Session Flow](#authentication--session-flow)
4. [Permission Validation Architecture](#permission-validation-architecture)
5. [Chat Message Flow](#chat-message-flow)
6. [MCP Tool Execution Flow](#mcp-tool-execution-flow)
7. [Data Models & Schemas](#data-models--schemas)
8. [API Specifications](#api-specifications)
9. [Security Architecture](#security-architecture)
10. [Caching Strategy](#caching-strategy)
11. [Error Handling Architecture](#error-handling-architecture)
12. [Monitoring & Observability](#monitoring--observability)

---

## System Architecture Overview

### High-Level 3-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                            │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Clarity PPM Application                        │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │        Custom Page (HTML/CSS/JavaScript)             │  │ │
│  │  │                                                       │  │ │
│  │  │  ┌────────────────┐  ┌──────────────────────────┐   │  │ │
│  │  │  │  Chat UI       │  │  Session Manager         │   │  │ │
│  │  │  │  - Messages    │  │  - Token extraction      │   │  │ │
│  │  │  │  - Input       │  │  - Validation            │   │  │ │
│  │  │  │  - History     │  │  - Refresh               │   │  │ │
│  │  │  └────────────────┘  └──────────────────────────┘   │  │ │
│  │  │                                                       │  │ │
│  │  │  ┌─────────────────────────────────────────────────┐ │  │ │
│  │  │  │  Middleware API Client                          │ │  │ │
│  │  │  │  - HTTP/HTTPS requests                          │ │  │ │
│  │  │  │  - SSE streaming handler                        │ │  │ │
│  │  │  │  - Error handling                               │ │  │ │
│  │  │  └─────────────────────────────────────────────────┘ │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTPS + JWT Token
                           │ (Session-based authentication)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                             │
│                   (Middleware Server - AWS EC2)                  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │               API Gateway / Express Server                  │ │
│  │                                                             │ │
│  │  ┌──────────────┐  ┌───────────────┐  ┌────────────────┐  │ │
│  │  │ Auth Module  │  │  Rate Limiter │  │  CORS Config   │  │ │
│  │  │ - JWT verify │  │  - Per user   │  │  - Clarity only│  │ │
│  │  │ - Session    │  │  - Global     │  │  - Headers     │  │ │
│  │  └──────────────┘  └───────────────┘  └────────────────┘  │ │
│  │                                                             │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │            Request Router                             │  │ │
│  │  │  /api/auth/*    → Auth Controller                    │  │ │
│  │  │  /api/chat/*    → Chat Controller                    │  │ │
│  │  │  /api/health    → Health Check                       │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  │                                                             │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │             Business Logic Layer                      │  │ │
│  │  │                                                        │  │ │
│  │  │  ┌─────────────────┐    ┌─────────────────────────┐  │  │ │
│  │  │  │ Chat Service    │    │  Permission Service     │  │  │ │
│  │  │  │ - AI calls      │    │  - Validation           │  │  │ │
│  │  │  │ - MCP routing   │    │  - Caching              │  │  │ │
│  │  │  │ - Context mgmt  │    │  - OBS checks           │  │  │ │
│  │  │  └─────────────────┘    └─────────────────────────┘  │  │ │
│  │  │                                                        │  │ │
│  │  │  ┌─────────────────┐    ┌─────────────────────────┐  │  │ │
│  │  │  │ Clarity Service │    │  Conversation Service   │  │  │ │
│  │  │  │ - API client    │    │  - History mgmt         │  │  │ │
│  │  │  │ - Session valid │    │  - Storage              │  │  │ │
│  │  │  └─────────────────┘    └─────────────────────────┘  │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    MCP Server Process                       │ │
│  │                                                             │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │  MCP Protocol Handler                                 │  │ │
│  │  │  - Tool registration                                  │  │ │
│  │  │  - Tool execution                                     │  │ │
│  │  │  - Error handling                                     │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  │                                                             │ │
│  │  ┌────────────────────────────────────────────────────┐    │ │
│  │  │            MCP Tools (15 tools)                    │    │ │
│  │  │                                                     │    │ │
│  │  │  ┌───────────────┐  ┌──────────────────────────┐  │    │ │
│  │  │  │ Project Tools │  │  Task Tools              │  │    │ │
│  │  │  │ - Get         │  │  - Get / Create          │  │    │ │
│  │  │  │ - List        │  │  - Update / Search       │  │    │ │
│  │  │  │ - Update      │  │  - Assign / Dependencies │  │    │ │
│  │  │  └───────────────┘  └──────────────────────────┘  │    │ │
│  │  │                                                     │    │ │
│  │  │  ┌───────────────┐  ┌──────────────────────────┐  │    │ │
│  │  │  │Resource Tools │  │  Timesheet Tools         │  │    │ │
│  │  │  │ - Availability│  │  - Get / Update          │  │    │ │
│  │  │  │ - Allocation  │  │  - Actuals               │  │    │ │
│  │  │  └───────────────┘  └──────────────────────────┘  │    │ │
│  │  └────────────────────────────────────────────────────┘    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  Cache Layer (Redis)                        │ │
│  │  - User permissions (TTL: 15 min)                           │ │
│  │  - Session validation (TTL: 5 min)                          │ │
│  │  - Project data (TTL: 2 min)                                │ │
│  │  - Task data (TTL: 1 min)                                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Database (PostgreSQL)                          │ │
│  │  - Conversations                                            │ │
│  │  - Messages                                                 │ │
│  │  - Audit logs                                               │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────┬───────────────────────────────────────┘
                           │ API Calls (HTTPS)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    INTEGRATION LAYER                             │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │ Clarity REST API │  │  Claude API      │  │  CloudWatch  │  │
│  │  - /projects     │  │  - Messages      │  │  - Logs      │  │
│  │  - /tasks        │  │  - MCP tools     │  │  - Metrics   │  │
│  │  - /resources    │  │  - Streaming     │  │  - Alarms    │  │
│  │  - /timesheets   │  │                  │  │              │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

**Presentation Layer:**
- User interface rendering
- User input collection
- Session management (client-side)
- Real-time message display with streaming
- Error display and user feedback

**Application Layer (Middleware):**
- Authentication and authorization
- Request routing and validation
- Business logic orchestration
- MCP tool execution
- Rate limiting and throttling
- Caching strategy implementation
- Conversation persistence

**Application Layer (MCP Server):**
- MCP protocol implementation
- Tool registration and management
- Clarity API integration
- Permission-aware tool execution
- Error handling and logging

**Integration Layer:**
- External API communication
- Data transformation
- Monitoring and logging
- Alert management

---

## AWS Deployment Architecture

### Infrastructure Layout

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          AWS REGION (eu-west-1)                          │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                           VPC (10.0.0.0/16)                         │ │
│  │                                                                     │ │
│  │  ┌───────────────────────────────────────────────────────────────┐ │ │
│  │  │        Public Subnet (10.0.1.0/24) - AZ-A                     │ │ │
│  │  │                                                                │ │ │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │ │
│  │  │  │         Application Load Balancer (ALB)                  │ │ │ │
│  │  │  │  - HTTPS (443) → Middleware                              │ │ │ │
│  │  │  │  - SSL/TLS termination                                   │ │ │ │
│  │  │  │  - Health checks                                         │ │ │ │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │ │
│  │  │                          ↓                                     │ │ │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │ │
│  │  │  │         AWS WAF (Web Application Firewall)               │ │ │ │
│  │  │  │  - SQL injection protection                              │ │ │ │
│  │  │  │  - XSS protection                                        │ │ │ │
│  │  │  │  - Rate limiting rules                                   │ │ │ │
│  │  │  │  - IP whitelist (optional)                               │ │ │ │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │ │
│  │  │                                                                │ │ │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │ │
│  │  │  │     NAT Gateway (for private subnet outbound)            │ │ │ │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │ │
│  │  └───────────────────────────────────────────────────────────────┘ │ │
│  │                                                                     │ │
│  │  ┌───────────────────────────────────────────────────────────────┐ │ │
│  │  │      Private Subnet (10.0.10.0/24) - AZ-A                     │ │ │
│  │  │                                                                │ │ │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │ │
│  │  │  │  EC2 Instance: Middleware + MCP Server                   │ │ │ │
│  │  │  │  ┌────────────────────────────────────────────────────┐  │ │ │ │
│  │  │  │  │  OS: Ubuntu 22.04 LTS                              │  │ │ │ │
│  │  │  │  │  Type: t3.medium (2 vCPU, 4GB RAM)                 │  │ │ │ │
│  │  │  │  │  Storage: 50GB gp3 SSD                             │  │ │ │ │
│  │  │  │  │                                                     │  │ │ │ │
│  │  │  │  │  Services:                                         │  │ │ │ │
│  │  │  │  │  ├─ Node.js 20.x LTS                              │  │ │ │ │
│  │  │  │  │  ├─ Express Server (Port 3000)                    │  │ │ │ │
│  │  │  │  │  ├─ MCP Server Process                            │  │ │ │ │
│  │  │  │  │  ├─ PM2 Process Manager                           │  │ │ │ │
│  │  │  │  │  └─ CloudWatch Agent                              │  │ │ │ │
│  │  │  │  └────────────────────────────────────────────────────┘  │ │ │ │
│  │  │  │                                                          │ │ │ │
│  │  │  │  Security Group: SG-Middleware                           │ │ │ │
│  │  │  │  Inbound:                                                │ │ │ │
│  │  │  │   - Port 3000 from ALB only                              │ │ │ │
│  │  │  │   - Port 22 from Bastion host only                       │ │ │ │
│  │  │  │  Outbound:                                               │ │ │ │
│  │  │  │   - Port 443 to internet (via NAT)                       │ │ │ │
│  │  │  │   - Port 6379 to Redis                                   │ │ │ │
│  │  │  │   - Port 5432 to PostgreSQL                              │ │ │ │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │ │
│  │  │                                                                │ │ │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │ │
│  │  │  │  ElastiCache: Redis Cluster                              │ │ │ │
│  │  │  │  ┌────────────────────────────────────────────────────┐  │ │ │ │
│  │  │  │  │  Type: cache.t3.micro (0.5GB)                      │  │ │ │ │
│  │  │  │  │  Engine: Redis 7.x                                 │  │ │ │ │
│  │  │  │  │  Encrypted: Yes (at rest and in transit)           │  │ │ │ │
│  │  │  │  │  Multi-AZ: Yes (for HA)                            │  │ │ │ │
│  │  │  │  │                                                     │  │ │ │ │
│  │  │  │  │  Data stored:                                      │  │ │ │ │
│  │  │  │  │  - Session validation cache                        │  │ │ │ │
│  │  │  │  │  - User permissions cache                          │  │ │ │ │
│  │  │  │  │  - Project/Task data cache                         │  │ │ │ │
│  │  │  │  │  - Rate limiting counters                          │  │ │ │ │
│  │  │  │  └────────────────────────────────────────────────────┘  │ │ │ │
│  │  │  │                                                          │ │ │ │
│  │  │  │  Security Group: SG-Redis                                │ │ │ │
│  │  │  │  Inbound:                                                │ │ │ │
│  │  │  │   - Port 6379 from SG-Middleware only                    │ │ │ │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │ │
│  │  │                                                                │ │ │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │ │
│  │  │  │  RDS: PostgreSQL Database                                │ │ │ │
│  │  │  │  ┌────────────────────────────────────────────────────┐  │ │ │ │
│  │  │  │  │  Type: db.t3.micro (1 vCPU, 1GB RAM)               │  │ │ │ │
│  │  │  │  │  Engine: PostgreSQL 15.x                           │  │ │ │ │
│  │  │  │  │  Storage: 20GB gp3 SSD (autoscaling enabled)       │  │ │ │ │
│  │  │  │  │  Encrypted: Yes                                    │  │ │ │ │
│  │  │  │  │  Multi-AZ: Yes (for HA)                            │  │ │ │ │
│  │  │  │  │  Backup: Daily automated snapshots (7-day retain)  │  │ │ │ │
│  │  │  │  │                                                     │  │ │ │ │
│  │  │  │  │  Databases:                                        │  │ │ │ │
│  │  │  │  │  - clarity_chatbot (main)                          │  │ │ │ │
│  │  │  │  │                                                     │  │ │ │ │
│  │  │  │  │  Tables:                                           │  │ │ │ │
│  │  │  │  │  - conversations                                   │  │ │ │ │
│  │  │  │  │  - messages                                        │  │ │ │ │
│  │  │  │  │  - audit_logs                                      │  │ │ │ │
│  │  │  │  └────────────────────────────────────────────────────┘  │ │ │ │
│  │  │  │                                                          │ │ │ │
│  │  │  │  Security Group: SG-PostgreSQL                           │ │ │ │
│  │  │  │  Inbound:                                                │ │ │ │
│  │  │  │   - Port 5432 from SG-Middleware only                    │ │ │ │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │ │
│  │  └───────────────────────────────────────────────────────────────┘ │ │
│  │                                                                     │ │
│  │  ┌───────────────────────────────────────────────────────────────┐ │ │
│  │  │     Public Subnet (10.0.2.0/24) - AZ-A                        │ │ │
│  │  │                                                                │ │ │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │ │
│  │  │  │  Bastion Host (EC2)                                      │ │ │ │
│  │  │  │  - For SSH access to private instances                  │ │ │ │
│  │  │  │  - Type: t3.nano                                        │ │ │ │
│  │  │  │  - Only accessible from specific IPs                    │ │ │ │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │ │
│  │  └───────────────────────────────────────────────────────────────┘ │ │
│  │                                                                     │ │
│  │  ┌───────────────────────────────────────────────────────────────┐ │ │
│  │  │        Existing: Clarity PPM Server (Separate Network)        │ │ │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │ │
│  │  │  │  EC2 Windows Server                                      │ │ │ │
│  │  │  │  - Clarity PPM Application                               │ │ │ │
│  │  │  │  - Custom Page accessible                                │ │ │ │
│  │  │  │  - Outbound HTTPS to Middleware                          │ │ │ │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │ │
│  │  └───────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                      AWS Services Used                              │ │
│  │                                                                     │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐   │ │
│  │  │  CloudWatch     │  │ Secrets Manager │  │  Systems Manager │   │ │
│  │  │  - Logs         │  │  - API keys     │  │  - Parameter     │   │ │
│  │  │  - Metrics      │  │  - DB passwords │  │    Store         │   │ │
│  │  │  - Alarms       │  │  - JWT secrets  │  │  - Patching      │   │ │
│  │  │  - Dashboards   │  └─────────────────┘  └──────────────────┘   │ │
│  │  └─────────────────┘                                               │ │
│  │                                                                     │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐   │ │
│  │  │  IAM            │  │  Route 53       │  │  Certificate     │   │ │
│  │  │  - Roles        │  │  - DNS          │  │  Manager (ACM)   │   │ │
│  │  │  - Policies     │  │  - Health check │  │  - SSL certs     │   │ │
│  │  └─────────────────┘  └─────────────────┘  └──────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘

                                     ↕
                         Internet Gateway (0.0.0.0/0)
                                     ↕
                   ┌──────────────────────────────────┐
                   │  External Services               │
                   │  - Anthropic Claude API          │
                   │  - Clarity REST API (if external)│
                   └──────────────────────────────────┘
```

### Network Flow

```
User Browser (Clarity Page)
         ↓ HTTPS
    Internet Gateway
         ↓
    Application Load Balancer (SSL termination)
         ↓
    AWS WAF (security filtering)
         ↓
    Middleware EC2 (Private Subnet)
         ↓
    ├─→ Redis (Cache queries)
    ├─→ PostgreSQL (Conversation storage)
    ├─→ MCP Server (Tool execution)
    └─→ External APIs (Claude, Clarity)
```

### High Availability & Disaster Recovery

**Availability Strategy:**
```
┌─────────────────────────────────────────────┐
│  Availability Zone A        AZ B            │
│                                             │
│  ┌────────────────┐    ┌────────────────┐  │
│  │  Middleware    │    │  Middleware    │  │
│  │  EC2 Primary   │    │  EC2 Standby   │  │
│  └────────────────┘    └────────────────┘  │
│          ↓                     ↑             │
│  ┌────────────────────────────────────────┐ │
│  │    Application Load Balancer           │ │
│  │    (Auto failover to healthy instance) │ │
│  └────────────────────────────────────────┘ │
│                                             │
│  ┌────────────────┐    ┌────────────────┐  │
│  │  Redis Primary │→→→→│  Redis Replica │  │
│  └────────────────┘    └────────────────┘  │
│                                             │
│  ┌────────────────┐    ┌────────────────┐  │
│  │  RDS Primary   │→→→→│  RDS Standby   │  │
│  └────────────────┘    └────────────────┘  │
└─────────────────────────────────────────────┘

Auto-failover: < 2 minutes
RPO (Recovery Point Objective): < 5 minutes
RTO (Recovery Time Objective): < 10 minutes
```

**Backup Strategy:**
- RDS automated daily snapshots (retained 7 days)
- Weekly manual snapshots (retained 30 days)
- Redis AOF (Append Only File) persistence enabled
- Configuration files in S3 with versioning
- Infrastructure as Code (Terraform/CloudFormation) in Git

---

## Authentication & Session Flow

### Complete Authentication Sequence

```
┌──────────┐                 ┌────────────┐                ┌────────────┐
│  User    │                 │  Clarity   │                │ Middleware │
│ Browser  │                 │    Page    │                │   Server   │
└────┬─────┘                 └─────┬──────┘                └─────┬──────┘
     │                             │                              │
     │  1. Navigate to             │                              │
     │     Clarity chatbot page    │                              │
     ├────────────────────────────>│                              │
     │                             │                              │
     │  2. Page loads              │                              │
     │<────────────────────────────┤                              │
     │                             │                              │
     │                             │  3. Initialize session       │
     │                             │     manager on page load     │
     │                             │─┐                            │
     │                             │ │                            │
     │                             │<┘                            │
     │                             │                              │
     │                             │  4. Extract Clarity session  │
     │                             │     (JSESSIONID cookie)      │
     │                             │─┐                            │
     │                             │ │ document.cookie           │
     │                             │<┘                            │
     │                             │                              │
     │                             │  5. Get user ID              │
     │                             │     (from Clarity context)   │
     │                             │─┐                            │
     │                             │ │ window.currentUserId      │
     │                             │<┘ or API call                │
     │                             │                              │
     │                             │  6. POST /api/auth/validate  │
     │                             │     {                        │
     │                             │       claritySessionId,      │
     │                             │       userId                 │
     │                             │     }                        │
     │                             ├─────────────────────────────>│
     │                             │                              │
     │                             │                              │  7. Check Redis cache
     │                             │                              │     for session
     │                             │                              │─┐
     │                             │                              │ │ GET session:{id}
     │                             │                              │<┘
     │                             │                              │
     │                             │                              │  8. If not in cache,
     │                             │                              │     validate with
     │                             │                              │     Clarity API
     │                             │                              │─┐
     │                             │                              │ │ GET /api/session/
     │                             │                              │ │     validate
     │                             │                              │<┘
     │                             │                ┌─────────────┤
     │                             │                │ Clarity API │
     │                             │                │   Server    │
     │                             │                └─────────────┤
     │                             │                              │<┐ 9. Fetch user
     │                             │                              │ │    permissions
     │                             │                              │ │    and groups
     │                             │                              │─┘
     │                             │                              │
     │                             │                              │  10. Cache session
     │                             │                              │      in Redis
     │                             │                              │─┐    (TTL: 5 min)
     │                             │                              │ │
     │                             │                              │<┘
     │                             │                              │
     │                             │                              │  11. Generate JWT
     │                             │                              │      for middleware
     │                             │                              │─┐    (exp: 30 min)
     │                             │                              │ │
     │                             │                              │<┘
     │                             │                              │
     │                             │  12. Return success          │
     │                             │      {                       │
     │                             │        middlewareToken: JWT, │
     │                             │        user: {...},          │
     │                             │        expiresIn: 1800       │
     │                             │      }                       │
     │                             │<─────────────────────────────┤
     │                             │                              │
     │                             │  13. Store token in memory   │
     │                             │      (NOT localStorage)      │
     │                             │─┐                            │
     │                             │ │                            │
     │                             │<┘                            │
     │                             │                              │
     │  14. Display ready UI       │                              │
     │<────────────────────────────┤                              │
     │                             │                              │
     │                             │                              │
     │  ═══════════ USER AUTHENTICATED ════════════               │
     │                             │                              │
     │                             │                              │
     │  15. User types message     │                              │
     ├────────────────────────────>│                              │
     │                             │                              │
     │                             │  16. POST /api/chat/message  │
     │                             │      Authorization:          │
     │                             │      Bearer {JWT}            │
     │                             ├─────────────────────────────>│
     │                             │                              │
     │                             │                              │  17. Verify JWT
     │                             │                              │─┐    signature
     │                             │                              │ │    and expiry
     │                             │                              │<┘
     │                             │                              │
     │                             │                              │  18. Extract userId
     │                             │                              │      from JWT
     │                             │                              │─┐    payload
     │                             │                              │ │
     │                             │                              │<┘
     │                             │                              │
     │                             │  19. Process request         │
     │                             │      (see Chat Flow)         │
     │                             │                              │
     │                             │                              │
     │  ═══════════ SESSION EXPIRY HANDLING ═══════════           │
     │                             │                              │
     │                             │  20. JWT expired (401)       │
     │                             │<─────────────────────────────┤
     │                             │                              │
     │                             │  21. Attempt auto-refresh    │
     │                             │      (re-validate Clarity    │
     │                             │       session)               │
     │                             ├─────────────────────────────>│
     │                             │                              │
     │                             │  22. New JWT or 401          │
     │                             │<─────────────────────────────┤
     │                             │                              │
     │  23. If 401: Show           │                              │
     │      re-login prompt        │                              │
     │<────────────────────────────┤                              │
     │                             │                              │
```

### Session Validation Logic

```typescript
// Middleware: Session validation function

interface SessionValidationRequest {
  claritySessionId: string;
  userId: string;
}

interface SessionValidationResponse {
  valid: boolean;
  middlewareToken?: string;
  user?: {
    id: string;
    name: string;
    email: string;
    groups: string[];
  };
  expiresIn?: number;
  error?: string;
}

async function validateSession(
  req: SessionValidationRequest
): Promise<SessionValidationResponse> {
  
  // 1. Check Redis cache first (fast path)
  const cachedSession = await redis.get(`session:${req.claritySessionId}`);
  if (cachedSession) {
    const session = JSON.parse(cachedSession);
    
    // Verify it's the correct user
    if (session.userId === req.userId) {
      // Generate new JWT for this request
      const jwt = generateJWT({
        userId: session.userId,
        userName: session.userName,
        groups: session.groups
      });
      
      return {
        valid: true,
        middlewareToken: jwt,
        user: {
          id: session.userId,
          name: session.userName,
          email: session.email,
          groups: session.groups
        },
        expiresIn: 1800 // 30 minutes
      };
    }
  }
  
  // 2. Cache miss - validate with Clarity (slow path)
  try {
    const clarityResponse = await axios.get(
      `${CLARITY_URL}/api/v1/session/validate`,
      {
        headers: {
          'Cookie': `JSESSIONID=${req.claritySessionId}`
        }
      }
    );
    
    if (clarityResponse.status !== 200) {
      return {
        valid: false,
        error: 'INVALID_SESSION'
      };
    }
    
    const clarityUser = clarityResponse.data;
    
    // 3. Fetch user permissions
    const permissions = await fetchUserPermissions(clarityUser.id);
    
    // 4. Cache the session
    await redis.setex(
      `session:${req.claritySessionId}`,
      300, // 5 minutes TTL
      JSON.stringify({
        userId: clarityUser.id,
        userName: clarityUser.name,
        email: clarityUser.email,
        groups: clarityUser.groups,
        permissions: permissions,
        validatedAt: Date.now()
      })
    );
    
    // 5. Generate JWT
    const jwt = generateJWT({
      userId: clarityUser.id,
      userName: clarityUser.name,
      groups: clarityUser.groups
    });
    
    return {
      valid: true,
      middlewareToken: jwt,
      user: {
        id: clarityUser.id,
        name: clarityUser.name,
        email: clarityUser.email,
        groups: clarityUser.groups
      },
      expiresIn: 1800
    };
    
  } catch (error) {
    logger.error('Session validation failed', { error, req });
    return {
      valid: false,
      error: 'VALIDATION_FAILED'
    };
  }
}

// JWT Generation
function generateJWT(payload: any): string {
  return jwt.sign(
    payload,
    JWT_SECRET,
    {
      expiresIn: '30m',
      issuer: 'clarity-chatbot-middleware',
      audience: 'clarity-chatbot'
    }
  );
}
```

---

## Permission Validation Architecture

### Permission Check Flow

```
┌────────────┐         ┌─────────────┐         ┌──────────────┐
│  MCP Tool  │         │ Permission  │         │  Redis       │
│  Request   │         │ Validator   │         │  Cache       │
└─────┬──────┘         └──────┬──────┘         └──────┬───────┘
      │                       │                       │
      │  1. Tool execution    │                       │
      │     request with      │                       │
      │     userId, resource  │                       │
      ├──────────────────────>│                       │
      │                       │                       │
      │                       │  2. Check cache       │
      │                       │     for permissions   │
      │                       ├──────────────────────>│
      │                       │                       │
      │                       │  3. Cache hit?        │
      │                       │<──────────────────────┤
      │                       │     (permissions)     │
      │                       │                       │
      │                       │                       │
      │                   [IF CACHE MISS]             │
      │                       │                       │
      │                       │  4. Fetch from        │
      │                       │     Clarity API       │
      │                       ├──────────────────────────────┐
      │                       │                              │
      │                       │  GET /users/{id}/rights      │
      │                       │  GET /users/{id}/groups      │
      │                       │  GET /users/{id}/obs-access  │
      │                       │                              │
      │                       │<─────────────────────────────┤
      │                       │  5. Parse permissions        │
      │                       │                              │
      │                       │  6. Cache for 15 min         │
      │                       ├─────────────────────────────>│
      │                       │                              │
      │                       │                              │
      │                   [PERMISSION CHECK]                 │
      │                       │                              │
      │                       │  7. Validate global rights   │
      │                       │─┐   (e.g., project.task.edit)│
      │                       │ │                            │
      │                       │<┘                            │
      │                       │                              │
      │                       │  8. Validate OBS access      │
      │                       │─┐   (Organizational)         │
      │                       │ │                            │
      │                       │<┘                            │
      │                       │                              │
      │                       │  9. Validate instance access │
      │                       │─┐   (specific resource)      │
      │                       │ │                            │
      │                       │<┘                            │
      │                       │                              │
      │                       │  10. Decision: ALLOW/DENY    │
      │                       │                              │
      │  11. Return result    │                              │
      │      {                │                              │
      │        allowed: bool, │                              │
      │        reason: string │                              │
      │      }                │                              │
      │<──────────────────────┤                              │
      │                       │                              │
      │  12a. If ALLOWED:     │                              │
      │       Execute tool    │                              │
      │                       │                              │
      │  12b. If DENIED:      │                              │
      │       Return error    │                              │
      │       with reason     │                              │
      │                       │                              │
```

### Permission Model

```typescript
interface UserPermissions {
  userId: string;
  globalRights: GlobalRight[];
  groups: Group[];
  obsAccess: OBSAccess[];
  instanceRights: InstanceRight[];
  cachedAt: number;
  expiresAt: number;
}

interface GlobalRight {
  id: string;
  name: string; // e.g., "project.task.edit"
  granted: boolean;
}

interface Group {
  id: string;
  name: string;
  rights: string[];
}

interface OBSAccess {
  obsType: string; // "department", "location", etc.
  obsId: string;
  accessLevel: "read" | "write" | "none";
}

interface InstanceRight {
  resourceType: "project" | "task" | "resource";
  resourceId: string;
  rights: string[]; // ["read", "write", "delete"]
}

// Permission check implementation
class PermissionValidator {
  
  async checkAccess(
    userId: string,
    operation: string, // "read" | "write" | "delete"
    resourceType: string,
    resourceId: string
  ): Promise<PermissionCheckResult> {
    
    // 1. Get user permissions (from cache or Clarity)
    const permissions = await this.getUserPermissions(userId);
    
    // 2. Build required right string
    const requiredRight = `${resourceType}.${operation}`;
    
    // 3. Check global rights
    const hasGlobalRight = permissions.globalRights.some(
      right => right.name === requiredRight && right.granted
    );
    
    if (!hasGlobalRight) {
      return {
        allowed: false,
        reason: `User lacks global right: ${requiredRight}`,
        details: {
          userId,
          operation,
          resourceType,
          resourceId
        }
      };
    }
    
    // 4. Check OBS access (if resource has OBS)
    const resource = await this.getResource(resourceType, resourceId);
    if (resource.obs) {
      const hasOBSAccess = this.checkOBSAccess(
        permissions.obsAccess,
        resource.obs,
        operation
      );
      
      if (!hasOBSAccess) {
        return {
          allowed: false,
          reason: `User lacks OBS access for ${resource.obs.type}`,
          details: {
            userId,
            obs: resource.obs,
            operation
          }
        };
      }
    }
    
    // 5. Check instance rights (specific resource permissions)
    const hasInstanceRight = await this.checkInstanceAccess(
      userId,
      resourceType,
      resourceId,
      operation
    );
    
    if (!hasInstanceRight) {
      return {
        allowed: false,
        reason: `User lacks access to specific ${resourceType}: ${resourceId}`,
        details: {
          userId,
          resourceType,
          resourceId,
          operation
        }
      };
    }
    
    // All checks passed
    return {
      allowed: true,
      reason: 'All permission checks passed'
    };
  }
  
  private async checkInstanceAccess(
    userId: string,
    resourceType: string,
    resourceId: string,
    operation: string
  ): Promise<boolean> {
    
    // Try to fetch the resource using user's credentials
    // If Clarity returns 403, user doesn't have access
    try {
      const response = await axios.get(
        `${CLARITY_URL}/api/v1/${resourceType}s/${resourceId}`,
        {
          headers: {
            'X-Clarity-User-ID': userId,
            'Authorization': `Bearer ${CLARITY_SERVICE_TOKEN}`
          }
        }
      );
      
      // Check if user can perform the operation
      if (operation === 'write' || operation === 'delete') {
        // For write/delete, check if resource is editable
        return response.data.editable === true;
      }
      
      // For read, if we got here, user has access
      return true;
      
    } catch (error) {
      if (error.response?.status === 403) {
        return false; // Access denied
      }
      throw error; // Other errors should be re-thrown
    }
  }
  
  private checkOBSAccess(
    userOBSAccess: OBSAccess[],
    resourceOBS: any,
    operation: string
  ): boolean {
    
    const requiredLevel = operation === 'read' ? 'read' : 'write';
    
    return userOBSAccess.some(obs => 
      obs.obsType === resourceOBS.type &&
      obs.obsId === resourceOBS.id &&
      (obs.accessLevel === requiredLevel || obs.accessLevel === 'write')
    );
  }
  
  private async getUserPermissions(userId: string): Promise<UserPermissions> {
    
    // Try cache first
    const cached = await redis.get(`permissions:${userId}`);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // Fetch from Clarity
    const [rights, groups, obsAccess] = await Promise.all([
      this.fetchGlobalRights(userId),
      this.fetchUserGroups(userId),
      this.fetchOBSAccess(userId)
    ]);
    
    const permissions: UserPermissions = {
      userId,
      globalRights: rights,
      groups,
      obsAccess,
      instanceRights: [], // Checked dynamically
      cachedAt: Date.now(),
      expiresAt: Date.now() + 900000 // 15 minutes
    };
    
    // Cache for 15 minutes
    await redis.setex(
      `permissions:${userId}`,
      900,
      JSON.stringify(permissions)
    );
    
    return permissions;
  }
}

interface PermissionCheckResult {
  allowed: boolean;
  reason: string;
  details?: any;
}
```

### Clarity Rights Hierarchy

```
Global Rights (from user groups)
    ↓
OBS (Organizational Breakdown Structure)
    ↓
Instance Rights (specific resource)

Example:
User wants to edit Task-123 in Project-456

1. ✓ Global Right: "project.task.edit" → PASS
2. ✓ OBS Access: Project-456 is in "Engineering" department, 
                  user has write access to "Engineering" → PASS
3. ✓ Instance Right: User is member of Project-456 → PASS

Result: ALLOWED
```

---

## Chat Message Flow

### End-to-End Message Flow with AI and MCP

```
┌────────┐    ┌──────────┐    ┌────────────┐    ┌──────────┐    ┌──────────┐
│  User  │    │ Frontend │    │ Middleware │    │ Claude   │    │   MCP    │
│        │    │          │    │            │    │   API    │    │  Server  │
└───┬────┘    └────┬─────┘    └──────┬─────┘    └────┬─────┘    └────┬─────┘
    │              │                 │                │               │
    │ 1. Type      │                 │                │               │
    │    message   │                 │                │               │
    ├─────────────>│                 │                │               │
    │              │                 │                │               │
    │              │ 2. POST /api/   │                │               │
    │              │    chat/message │                │               │
    │              │    + JWT        │                │               │
    │              ├────────────────>│                │               │
    │              │                 │                │               │
    │              │                 │ 3. Verify JWT  │               │
    │              │                 │─┐              │               │
    │              │                 │ │              │               │
    │              │                 │<┘              │               │
    │              │                 │                │               │
    │              │                 │ 4. Get user    │               │
    │              │                 │    context     │               │
    │              │                 │─┐              │               │
    │              │                 │ │              │               │
    │              │                 │<┘              │               │
    │              │                 │                │               │
    │              │                 │ 5. Build       │               │
    │              │                 │    conversation│               │
    │              │                 │    context     │               │
    │              │                 │─┐              │               │
    │              │                 │ │ Load history │               │
    │              │                 │<┘ from DB      │               │
    │              │                 │                │               │
    │              │                 │ 6. Call Claude │               │
    │              │                 │    API with    │               │
    │              │                 │    MCP config  │               │
    │              │                 ├───────────────>│               │
    │              │                 │                │               │
    │              │                 │                │ 7. Claude     │
    │              │                 │                │    processes  │
    │              │                 │                │    message    │
    │              │                 │                │─┐             │
    │              │                 │                │ │             │
    │              │                 │                │<┘             │
    │              │                 │                │               │
    │              │                 │                │ 8. Claude     │
    │              │                 │                │    needs tool │
    │              │                 │                │    (MCP call) │
    │              │                 │                ├──────────────>│
    │              │                 │                │               │
    │              │                 │  9. MCP receives tool request  │
    │              │                 │                │               │
    │              │                 │  ←──────────────────────────────┤
    │              │                 │  Tool: clarity_get_project      │
    │              │                 │  Input: {projectId: "PRJ-001"}  │
    │              │                 │                                 │
    │              │                 │<────────────────────────────────┤
    │              │                 │                │               │
    │              │                 │ 10. Validate   │               │
    │              │                 │     user       │               │
    │              │                 │     permission │               │
    │              │                 │─┐              │               │
    │              │                 │ │ Check if user│               │
    │              │                 │ │ can access   │               │
    │              │                 │ │ PRJ-001      │               │
    │              │                 │<┘              │               │
    │              │                 │                │               │
    │              │             [IF PERMISSION DENIED]               │
    │              │                 │                │               │
    │              │                 │────────────────────────────────>│
    │              │                 │  Error: Permission denied       │
    │              │                 │<────────────────────────────────┤
    │              │                 │                │               │
    │              │                 │                │<──────────────┤
    │              │                 │<───────────────┤  Tool error   │
    │              │                 │                │               │
    │              │<────────────────┤                │               │
    │              │  Error message  │                │               │
    │<─────────────┤                 │                │               │
    │  "You don't  │                 │                │               │
    │   have access│                 │                │               │
    │   to PRJ-001"│                 │                │               │
    │              │                 │                │               │
    │          [IF PERMISSION ALLOWED]                │               │
    │              │                 │                │               │
    │              │                 │ 11. Call       │               │
    │              │                 │     Clarity API│               │
    │              │                 ├────────────────────────────────┐
    │              │                 │                │               │
    │              │                 │  GET /api/v1/projects/PRJ-001  │
    │              │                 │                                │
    │              │                 │<───────────────────────────────┤
    │              │                 │  {name, status, dates, ...}    │
    │              │                 │                                │
    │              │                 │────────────────────────────────>│
    │              │                 │  Return project data            │
    │              │                 │                │               │
    │              │                 │                │<──────────────┤
    │              │                 │                │  Tool result  │
    │              │                 │                │               │
    │              │                 │ 12. Claude     │               │
    │              │                 │     continues  │               │
    │              │                 │     with result│               │
    │              │                 │                │─┐             │
    │              │                 │                │ │ Generate    │
    │              │                 │                │ │ response    │
    │              │                 │                │<┘             │
    │              │                 │                │               │
    │              │                 │ 13. Stream     │               │
    │              │                 │     response   │               │
    │              │                 │     tokens     │               │
    │              │                 │<───────────────┤               │
    │              │                 │  "Project      │               │
    │              │                 │   PRJ-001 is..." │             │
    │              │                 │                │               │
    │              │ 14. SSE stream  │                │               │
    │              │     to frontend │                │               │
    │              │<────────────────┤                │               │
    │              │  data: {token}  │                │               │
    │              │                 │                │               │
    │ 15. Display  │                 │                │               │
    │     response │                 │                │               │
    │     in real  │                 │                │               │
    │     time     │                 │                │               │
    │<─────────────┤                 │                │               │
    │  "Project..."│                 │                │               │
    │              │                 │                │               │
    │              │                 │ 16. Save       │               │
    │              │                 │     conversation               │
    │              │                 │     to DB      │               │
    │              │                 │─┐              │               │
    │              │                 │ │              │               │
    │              │                 │<┘              │               │
    │              │                 │                │               │
    │              │ 17. Complete    │                │               │
    │              │<────────────────┤                │               │
    │              │  data: {done}   │                │               │
    │              │                 │                │               │
```

### SSE (Server-Sent Events) Stream Format

```javascript
// Client-side: SSE connection
const eventSource = new EventSource('/api/chat/stream?token=' + jwtToken);

eventSource.addEventListener('message', (event) => {
  const data = JSON.parse(event.data);
  
  switch (data.type) {
    case 'token':
      // Append token to current message
      appendToMessage(data.content);
      break;
      
    case 'tool_use':
      // Show tool usage indicator
      showToolIndicator(data.tool, data.input);
      break;
      
    case 'tool_result':
      // Show tool result (optional)
      showToolResult(data.result);
      break;
      
    case 'error':
      // Display error
      showError(data.message);
      break;
      
    case 'done':
      // Streaming complete
      markMessageComplete();
      eventSource.close();
      break;
  }
});

// Server-side: SSE response
function streamChatResponse(res, claudeStream) {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive'
  });
  
  claudeStream.on('text', (text) => {
    res.write(`data: ${JSON.stringify({
      type: 'token',
      content: text
    })}\n\n`);
  });
  
  claudeStream.on('tool_use', (tool) => {
    res.write(`data: ${JSON.stringify({
      type: 'tool_use',
      tool: tool.name,
      input: tool.input
    })}\n\n`);
  });
  
  claudeStream.on('tool_result', (result) => {
    res.write(`data: ${JSON.stringify({
      type: 'tool_result',
      result: result
    })}\n\n`);
  });
  
  claudeStream.on('error', (error) => {
    res.write(`data: ${JSON.stringify({
      type: 'error',
      message: error.message
    })}\n\n`);
  });
  
  claudeStream.on('end', () => {
    res.write(`data: ${JSON.stringify({
      type: 'done'
    })}\n\n`);
    res.end();
  });
}
```

---

## MCP Tool Execution Flow

### Detailed Tool Execution with Permission Validation

```
User Message: "Show me tasks in Project Tokyo"
                        ↓
            Claude API processes message
                        ↓
        Claude decides to use MCP tool
                        ↓
┌─────────────────────────────────────────────┐
│  MCP Tool Call from Claude                  │
│                                             │
│  Tool: clarity_search_tasks                 │
│  Input: {                                   │
│    projectId: "tokyo-2025",                 │
│    status: "active"                         │
│  }                                          │
│  Context: {                                 │
│    userId: "user123" (from middleware)      │
│  }                                          │
└───────────────┬─────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────┐
│  MCP Server: Tool Handler                   │
│                                             │
│  1. Receive tool call                       │
│  2. Extract userId from context             │
│  3. Validate input schema                   │
└───────────────┬─────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────┐
│  Permission Validation                      │
│                                             │
│  Check if user123 can:                      │
│   ✓ Read project "tokyo-2025"              │
│   ✓ List tasks in project                  │
└───────────────┬─────────────────────────────┘
                │
        ┌───────┴──────┐
        │              │
    ALLOWED         DENIED
        │              │
        ▼              ▼
  Execute Tool    Return Error
        │              │
        ▼              │
┌──────────────┐       │
│ Clarity API  │       │
│              │       │
│ GET /tasks?  │       │
│ project=...  │       │
│ status=...   │       │
└──────┬───────┘       │
       │               │
       ▼               ▼
  Parse Result    Format Error
       │               │
       ▼               ▼
┌────────────────────────────────┐
│  Return to Claude              │
│                                │
│  Success:                      │
│  {                             │
│    tasks: [                    │
│      {id, name, status, ...},  │
│      ...                       │
│    ]                           │
│  }                             │
│                                │
│  OR                            │
│                                │
│  Error:                        │
│  {                             │
│    error: "PERMISSION_DENIED", │
│    message: "User lacks..."    │
│  }                             │
└────────────────────────────────┘
```

### MCP Tool Implementation Template

```typescript
// MCP Tool: clarity_search_tasks

import { CallToolRequestSchema } from "@modelcontextprotocol/sdk/types.js";

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  
  if (name === "clarity_search_tasks") {
    try {
      // 1. Extract user context (injected by middleware)
      const userId = args._userId;
      const requestId = args._requestId;
      
      // 2. Validate input schema
      const input = TaskSearchInputSchema.parse(args);
      
      // 3. Log the request
      logger.info('Tool execution started', {
        requestId,
        userId,
        tool: 'clarity_search_tasks',
        input
      });
      
      // 4. Permission check
      const canReadProject = await permissionValidator.checkAccess(
        userId,
        'read',
        'project',
        input.projectId
      );
      
      if (!canReadProject.allowed) {
        logger.warn('Permission denied', {
          requestId,
          userId,
          tool: 'clarity_search_tasks',
          reason: canReadProject.reason
        });
        
        return {
          content: [
            {
              type: "text",
              text: JSON.stringify({
                error: "PERMISSION_DENIED",
                message: `You don't have permission to view tasks in this project.`,
                reason: canReadProject.reason
              })
            }
          ],
          isError: true
        };
      }
      
      // 5. Execute Clarity API call
      const tasks = await clarityClient.searchTasks({
        projectId: input.projectId,
        status: input.status,
        assignedTo: input.assignedTo,
        startDate: input.startDate,
        endDate: input.endDate
      });
      
      // 6. Filter tasks by permissions (user might not have access to all)
      const accessibleTasks = await filterTasksByPermission(
        tasks,
        userId
      );
      
      // 7. Format response
      const response = {
        projectId: input.projectId,
        taskCount: accessibleTasks.length,
        tasks: accessibleTasks.map(task => ({
          id: task.id,
          name: task.name,
          status: task.status,
          percentComplete: task.percentComplete,
          startDate: task.start,
          finishDate: task.finish,
          assignee: task.assignedTo,
          priority: task.priority
        }))
      };
      
      // 8. Log success
      logger.info('Tool execution completed', {
        requestId,
        userId,
        tool: 'clarity_search_tasks',
        resultCount: accessibleTasks.length
      });
      
      // 9. Return formatted result
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(response, null, 2)
          }
        ]
      };
      
    } catch (error) {
      // Error handling
      logger.error('Tool execution failed', {
        requestId,
        userId,
        tool: 'clarity_search_tasks',
        error: error.message,
        stack: error.stack
      });
      
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify({
              error: "EXECUTION_FAILED",
              message: "Failed to search tasks. Please try again.",
              details: process.env.NODE_ENV === 'development' ? error.message : undefined
            })
          }
        ],
        isError: true
      };
    }
  }
});

// Helper: Filter tasks by permission
async function filterTasksByPermission(
  tasks: Task[],
  userId: string
): Promise<Task[]> {
  const accessChecks = await Promise.all(
    tasks.map(task => 
      permissionValidator.checkAccess(userId, 'read', 'task', task.id)
    )
  );
  
  return tasks.filter((task, index) => accessChecks[index].allowed);
}
```

### Complete List of MCP Tools

```typescript
// All 15 MCP Tools for Clarity PPM

export const CLARITY_MCP_TOOLS = [
  {
    name: "clarity_get_project",
    description: "Retrieves detailed information about a specific Clarity PPM project including status, dates, budget, team members, and progress",
    inputSchema: {
      type: "object",
      properties: {
        projectId: {
          type: "string",
          description: "Unique identifier of the project"
        },
        includeFinancials: {
          type: "boolean",
          description: "Include budget and cost information (requires financial access)",
          default: false
        },
        includeTeam: {
          type: "boolean",
          description: "Include team member assignments",
          default: true
        }
      },
      required: ["projectId"]
    }
  },
  
  {
    name: "clarity_list_projects",
    description: "Lists projects accessible to the user with optional filtering by status, date range, or manager",
    inputSchema: {
      type: "object",
      properties: {
        status: {
          type: "string",
          enum: ["active", "on-hold", "completed", "cancelled", "all"],
          default: "active"
        },
        startDateFrom: {
          type: "string",
          format: "date",
          description: "Filter projects starting after this date (YYYY-MM-DD)"
        },
        managerId: {
          type: "string",
          description: "Filter by project manager ID"
        },
        limit: {
          type: "integer",
          description: "Maximum number of projects to return",
          default: 50,
          maximum: 200
        }
      }
    }
  },
  
  {
    name: "clarity_update_project",
    description: "Updates fields of an existing project such as status, dates, description, or manager",
    inputSchema: {
      type: "object",
      properties: {
        projectId: {
          type: "string"
        },
        updates: {
          type: "object",
          properties: {
            name: { type: "string" },
            description: { type: "string" },
            status: {
              type: "string",
              enum: ["active", "on-hold", "completed", "cancelled"]
            },
            startDate: {
              type: "string",
              format: "date"
            },
            finishDate: {
              type: "string",
              format: "date"
            },
            managerId: {
              type: "string"
            }
          }
        }
      },
      required: ["projectId", "updates"]
    }
  },
  
  {
    name: "clarity_get_task",
    description: "Retrieves detailed information about a specific task including assignments, dates, progress, and dependencies",
    inputSchema: {
      type: "object",
      properties: {
        taskId: {
          type: "string",
          description: "Unique identifier of the task"
        },
        includePredecessors: {
          type: "boolean",
          description: "Include predecessor tasks",
          default: true
        },
        includeSuccessors: {
          type: "boolean",
          description: "Include successor tasks",
          default: true
        }
      },
      required: ["taskId"]
    }
  },
  
  {
    name: "clarity_create_task",
    description: "Creates a new task in a project with specified properties like name, dates, assignee, and position in WBS",
    inputSchema: {
      type: "object",
      properties: {
        projectId: {
          type: "string"
        },
        name: {
          type: "string"
        },
        startDate: {
          type: "string",
          format: "date"
        },
        finishDate: {
          type: "string",
          format: "date"
        },
        assignedTo: {
          type: "string",
          description: "Resource ID to assign"
        },
        parentTaskId: {
          type: "string",
          description: "Parent task ID for WBS hierarchy (optional)"
        },
        duration: {
          type: "number",
          description: "Duration in days"
        },
        etc: {
          type: "number",
          description: "Estimate to complete (hours)"
        }
      },
      required: ["projectId", "name"]
    }
  },
  
  {
    name: "clarity_update_task",
    description: "Updates properties of an existing task such as dates, progress, assignments, or ETC",
    inputSchema: {
      type: "object",
      properties: {
        taskId: {
          type: "string"
        },
        updates: {
          type: "object",
          properties: {
            name: { type: "string" },
            startDate: {
              type: "string",
              format: "date"
            },
            finishDate: {
              type: "string",
              format: "date"
            },
            percentComplete: {
              type: "number",
              minimum: 0,
              maximum: 100
            },
            etc: {
              type: "number",
              description: "Estimate to complete (hours)"
            },
            assignedTo: {
              type: "string"
            },
            status: {
              type: "string",
              enum: ["not-started", "in-progress", "completed", "on-hold"]
            }
          }
        }
      },
      required: ["taskId", "updates"]
    }
  },
  
  {
    name: "clarity_search_tasks",
    description: "Searches for tasks across projects with filters for project, assignee, status, dates, and keywords",
    inputSchema: {
      type: "object",
      properties: {
        projectId: {
          type: "string",
          description: "Filter by specific project (optional)"
        },
        assignedTo: {
          type: "string",
          description: "Filter by assigned resource ID"
        },
        status: {
          type: "string",
          enum: ["not-started", "in-progress", "completed", "on-hold", "all"]
        },
        startDateFrom: {
          type: "string",
          format: "date"
        },
        startDateTo: {
          type: "string",
          format: "date"
        },
        keyword: {
          type: "string",
          description: "Search in task names and descriptions"
        },
        limit: {
          type: "integer",
          default: 50,
          maximum: 200
        }
      }
    }
  },
  
  {
    name: "clarity_create_dependency",
    description: "Creates a predecessor/successor dependency relationship between two tasks with lag/lead time",
    inputSchema: {
      type: "object",
      properties: {
        predecessorTaskId: {
          type: "string"
        },
        successorTaskId: {
          type: "string"
        },
        dependencyType: {
          type: "string",
          enum: ["FS", "FF", "SS", "SF"],
          description: "Finish-to-Start, Finish-to-Finish, Start-to-Start, Start-to-Finish",
          default: "FS"
        },
        lag: {
          type: "number",
          description: "Lag time in days (positive) or lead time (negative)",
          default: 0
        }
      },
      required: ["predecessorTaskId", "successorTaskId"]
    }
  },
  
  {
    name: "clarity_assign_resource",
    description: "Assigns a resource (user) to a task with specified allocation percentage and date range",
    inputSchema: {
      type: "object",
      properties: {
        taskId: {
          type: "string"
        },
        resourceId: {
          type: "string",
          description: "User/Resource ID to assign"
        },
        allocationPercent: {
          type: "number",
          description: "Allocation percentage (0-100)",
          minimum: 0,
          maximum: 100,
          default: 100
        },
        startDate: {
          type: "string",
          format: "date"
        },
        finishDate: {
          type: "string",
          format: "date"
        }
      },
      required: ["taskId", "resourceId"]
    }
  },
  
  {
    name: "clarity_get_resource_availability",
    description: "Checks resource availability and allocation for a specified date range",
    inputSchema: {
      type: "object",
      properties: {
        resourceId: {
          type: "string"
        },
        startDate: {
          type: "string",
          format: "date"
        },
        endDate: {
          type: "string",
          format: "date"
        }
      },
      required: ["resourceId", "startDate", "endDate"]
    }
  },
  
  {
    name: "clarity_get_timesheet",
    description: "Retrieves timesheet entries for a user, project, or date range showing actuals and remaining work",
    inputSchema: {
      type: "object",
      properties: {
        userId: {
          type: "string",
          description: "Filter by user (defaults to current user if not specified)"
        },
        projectId: {
          type: "string",
          description: "Filter by project"
        },
        startDate: {
          type: "string",
          format: "date"
        },
        endDate: {
          type: "string",
          format: "date"
        }
      },
      required: ["startDate", "endDate"]
    }
  },
  
  {
    name: "clarity_update_timesheet",
    description: "Adds or updates timesheet entries for tasks including actuals and ETC",
    inputSchema: {
      type: "object",
      properties: {
        taskId: {
          type: "string"
        },
        date: {
          type: "string",
          format: "date",
          description: "Date of the timesheet entry"
        },
        actualHours: {
          type: "number",
          description: "Hours worked on this date"
        },
        etc: {
          type: "number",
          description: "Estimate to complete (hours remaining)"
        },
        notes: {
          type: "string",
          description: "Optional notes about the work"
        }
      },
      required: ["taskId", "date"]
    }
  },
  
  {
    name: "clarity_get_project_financials",
    description: "Retrieves financial information for a project including budget, actuals, costs, and variance (requires financial access)",
    inputSchema: {
      type: "object",
      properties: {
        projectId: {
          type: "string"
        },
        includeResourceCosts: {
          type: "boolean",
          description: "Include breakdown by resource",
          default: false
        },
        includeForecast: {
          type: "boolean",
          description: "Include forecast data",
          default: true
        }
      },
      required: ["projectId"]
    }
  },
  
  {
    name: "clarity_generate_report",
    description: "Generates a predefined Clarity report with specified parameters and returns data or download link",
    inputSchema: {
      type: "object",
      properties: {
        reportType: {
          type: "string",
          enum: ["project-status", "resource-utilization", "timesheet-summary", "milestone-tracker", "risk-analysis"]
        },
        parameters: {
          type: "object",
          description: "Report-specific parameters"
        },
        format: {
          type: "string",
          enum: ["json", "csv", "pdf"],
          default: "json"
        }
      },
      required: ["reportType"]
    }
  },
  
  {
    name: "clarity_bulk_update_tasks",
    description: "Updates multiple tasks at once with the same or different property changes, with validation and rollback on errors",
    inputSchema: {
      type: "object",
      properties: {
        tasks: {
          type: "array",
          items: {
            type: "object",
            properties: {
              taskId: { type: "string" },
              updates: { type: "object" }
            },
            required: ["taskId", "updates"]
          }
        },
        validateOnly: {
          type: "boolean",
          description: "Validate changes without applying them",
          default: false
        }
      },
      required: ["tasks"]
    }
  }
];
```

---

## Data Models & Schemas

### Database Schema (PostgreSQL)

```sql
-- ===============================================
-- Conversations Table
-- ===============================================
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id VARCHAR(100) NOT NULL,
    title VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    message_count INTEGER DEFAULT 0,
    total_tokens INTEGER DEFAULT 0,
    deleted_at TIMESTAMP WITH TIME ZONE, -- Soft delete
    
    -- Indexes
    CONSTRAINT conversations_user_id_idx 
        CHECK (user_id IS NOT NULL AND user_id != '')
);

CREATE INDEX idx_conversations_user_id ON conversations(user_id) 
    WHERE deleted_at IS NULL;
CREATE INDEX idx_conversations_updated_at ON conversations(updated_at DESC) 
    WHERE deleted_at IS NULL;

-- ===============================================
-- Messages Table
-- ===============================================
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content TEXT NOT NULL,
    tokens_used INTEGER,
    tool_calls JSONB, -- Store MCP tool calls if any
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    -- Store metadata
    metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
CREATE INDEX idx_messages_created_at ON messages(created_at DESC);
CREATE INDEX idx_messages_tool_calls ON messages USING GIN (tool_calls) 
    WHERE tool_calls IS NOT NULL;

-- ===============================================
-- Audit Logs Table
-- ===============================================
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    user_id VARCHAR(100) NOT NULL,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50),
    resource_id VARCHAR(100),
    details JSONB,
    ip_address INET,
    user_agent TEXT,
    request_id VARCHAR(100),
    
    -- Result
    success BOOLEAN,
    error_message TEXT,
    
    -- Context
    conversation_id UUID REFERENCES conversations(id) ON DELETE SET NULL,
    message_id UUID REFERENCES messages(id) ON DELETE SET NULL
);

CREATE INDEX idx_audit_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_timestamp ON audit_logs(timestamp DESC);
CREATE INDEX idx_audit_action ON audit_logs(action);
CREATE INDEX idx_audit_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_conversation ON audit_logs(conversation_id) 
    WHERE conversation_id IS NOT NULL;

-- ===============================================
-- User Preferences Table (Optional)
-- ===============================================
CREATE TABLE user_preferences (
    user_id VARCHAR(100) PRIMARY KEY,
    preferences JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Example preferences:
-- {
--   "theme": "dark",
--   "language": "en",
--   "notifications": true,
--   "defaultProjectView": "active"
-- }

-- ===============================================
-- API Usage Tracking (for cost monitoring)
-- ===============================================
CREATE TABLE api_usage (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id VARCHAR(100) NOT NULL,
    conversation_id UUID REFERENCES conversations(id) ON DELETE SET NULL,
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    -- API details
    api_provider VARCHAR(50) NOT NULL, -- 'claude', 'clarity', etc.
    endpoint VARCHAR(200),
    
    -- Token usage (for AI APIs)
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    total_tokens INTEGER DEFAULT 0,
    
    -- Cost estimation
    estimated_cost DECIMAL(10, 6),
    
    -- Performance
    response_time_ms INTEGER,
    success BOOLEAN
);

CREATE INDEX idx_api_usage_user_id ON api_usage(user_id);
CREATE INDEX idx_api_usage_timestamp ON api_usage(timestamp DESC);
CREATE INDEX idx_api_usage_provider ON api_usage(api_provider);
CREATE INDEX idx_api_usage_conversation ON api_usage(conversation_id) 
    WHERE conversation_id IS NOT NULL;

-- ===============================================
-- Auto-update triggers
-- ===============================================

-- Update conversations.updated_at on message insert
CREATE OR REPLACE FUNCTION update_conversation_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE conversations 
    SET updated_at = CURRENT_TIMESTAMP,
        message_count = message_count + 1
    WHERE id = NEW.conversation_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_conversation_timestamp
AFTER INSERT ON messages
FOR EACH ROW
EXECUTE FUNCTION update_conversation_timestamp();

-- Update conversations.total_tokens
CREATE OR REPLACE FUNCTION update_conversation_tokens()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.tokens_used IS NOT NULL THEN
        UPDATE conversations 
        SET total_tokens = total_tokens + NEW.tokens_used
        WHERE id = NEW.conversation_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_conversation_tokens
AFTER INSERT ON messages
FOR EACH ROW
EXECUTE FUNCTION update_conversation_tokens();
```

### TypeScript Interfaces

```typescript
// ===============================================
// Core Data Types
// ===============================================

interface Conversation {
  id: string;
  userId: string;
  title: string | null;
  createdAt: Date;
  updatedAt: Date;
  messageCount: number;
  totalTokens: number;
  deletedAt: Date | null;
}

interface Message {
  id: string;
  conversationId: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  tokensUsed: number | null;
  toolCalls: ToolCall[] | null;
  createdAt: Date;
  metadata: Record<string, any>;
}

interface ToolCall {
  id: string;
  name: string;
  input: Record<string, any>;
  result?: any;
  error?: string;
  timestamp: Date;
  duration_ms: number;
}

interface AuditLog {
  id: string;
  timestamp: Date;
  userId: string;
  action: string;
  resourceType: string | null;
  resourceId: string | null;
  details: Record<string, any> | null;
  ipAddress: string | null;
  userAgent: string | null;
  requestId: string | null;
  success: boolean | null;
  errorMessage: string | null;
  conversationId: string | null;
  messageId: string | null;
}

interface ApiUsage {
  id: string;
  userId: string;
  conversationId: string | null;
  timestamp: Date;
  apiProvider: 'claude' | 'clarity' | 'other';
  endpoint: string | null;
  inputTokens: number;
  outputTokens: number;
  totalTokens: number;
  estimatedCost: number | null;
  responseTimeMs: number | null;
  success: boolean | null;
}

// ===============================================
// Clarity API Data Types
// ===============================================

interface ClarityProject {
  id: string;
  code: string;
  name: string;
  description: string | null;
  status: 'active' | 'on-hold' | 'completed' | 'cancelled';
  startDate: Date;
  finishDate: Date;
  actualStartDate: Date | null;
  actualFinishDate: Date | null;
  percentComplete: number;
  managerId: string;
  managerName: string;
  
  // Financial (if user has access)
  budget: number | null;
  cost: number | null;
  
  // OBS
  obs: {
    type: string;
    id: string;
    name: string;
  }[];
  
  // Team
  team: ProjectMember[];
  
  // Custom attributes
  customAttributes: Record<string, any>;
}

interface ClarityTask {
  id: string;
  projectId: string;
  name: string;
  status: 'not-started' | 'in-progress' | 'completed' | 'on-hold';
  startDate: Date;
  finishDate: Date;
  actualStartDate: Date | null;
  actualFinishDate: Date | null;
  percentComplete: number;
  duration: number; // days
  etc: number | null; // hours
  assignedTo: string | null;
  assigneeName: string | null;
  
  // WBS
  wbsSequence: string; // e.g., "1.2.3"
  parentTaskId: string | null;
  outline Outline: number;
  
  // Dependencies
  predecessors: TaskDependency[];
  successors: TaskDependency[];
  
  // Flags
  isMilestone: boolean;
  isSummaryTask: boolean;
  
  // Custom attributes
  customAttributes: Record<string, any>;
}

interface TaskDependency {
  taskId: string;
  taskName: string;
  type: 'FS' | 'FF' | 'SS' | 'SF';
  lag: number; // days
}

interface ProjectMember {
  resourceId: string;
  resourceName: string;
  role: string;
  allocationPercent: number;
  startDate: Date;
  endDate: Date;
}

interface TimesheetEntry {
  id: string;
  taskId: string;
  taskName: string;
  projectId: string;
  projectName: string;
  resourceId: string;
  date: Date;
  actualHours: number;
  etc: number | null;
  notes: string | null;
  approved: boolean;
}

interface ResourceAvailability {
  resourceId: string;
  resourceName: string;
  dateRange: {
    start: Date;
    end: Date;
  };
  totalCapacity: number; // hours
  allocatedHours: number;
  availableHours: number;
  allocationPercent: number;
  assignments: ResourceAssignment[];
}

interface ResourceAssignment {
  projectId: string;
  projectName: string;
  taskId: string | null;
  taskName: string | null;
  startDate: Date;
  endDate: Date;
  allocatedHours: number;
  allocationPercent: number;
}

// ===============================================
// API Request/Response Types
// ===============================================

interface ChatMessageRequest {
  message: string;
  conversationId?: string;
  stream?: boolean;
  context?: {
    projectId?: string;
    taskId?: string;
  };
}

interface ChatMessageResponse {
  conversationId: string;
  messageId: string;
  response: string;
  toolsUsed: ToolCall[];
  tokensUsed: number;
  timestamp: Date;
}

interface StreamChunk {
  type: 'token' | 'tool_use' | 'tool_result' | 'error' | 'done';
  content?: string;
  tool?: string;
  input?: any;
  result?: any;
  message?: string;
}

interface AuthValidationRequest {
  claritySessionId: string;
  userId: string;
}

interface AuthValidationResponse {
  success: boolean;
  middlewareToken?: string;
  user?: {
    id: string;
    name: string;
    email: string;
    groups: string[];
  };
  expiresIn?: number;
  error?: string;
}

interface PermissionCheckRequest {
  userId: string;
  operation: 'read' | 'write' | 'delete';
  resourceType: 'project' | 'task' | 'resource' | 'timesheet';
  resourceId: string;
}

interface PermissionCheckResponse {
  allowed: boolean;
  reason: string;
  details?: any;
}

// ===============================================
// Configuration Types
// ===============================================

interface AppConfig {
  // Server
  port: number;
  host: string;
  nodeEnv: 'development' | 'staging' | 'production';
  
  // Clarity
  clarityUrl: string;
  clarityApiToken: string;
  clarityTimeout: number; // ms
  
  // Claude API
  claudeApiKey: string;
  claudeModel: string;
  claudeMaxTokens: number;
  
  // Database
  databaseUrl: string;
  databasePoolMin: number;
  databasePoolMax: number;
  
  // Redis
  redisUrl: string;
  redisPassword: string | null;
  
  // JWT
  jwtSecret: string;
  jwtExpiration: string; // e.g., "30m"
  
  // Rate Limiting
  rateLimitPerUser: number; // requests per minute
  rateLimitGlobal: number; // requests per minute
  
  // Logging
  logLevel: 'debug' | 'info' | 'warn' | 'error';
  logFormat: 'json' | 'pretty';
  
  // CORS
  corsOrigins: string[];
  
  // Features
  enableConversationHistory: boolean;
  enableCaching: boolean;
  enableMetrics: boolean;
  
  // Cost Management
  maxTokensPerDay: number | null;
  maxCostPerDay: number | null; // USD
}
```

---

## API Specifications

### REST API Endpoints

```
Base URL: https://your-middleware.com/api
Authentication: Bearer token (JWT)
Content-Type: application/json
```

#### Authentication Endpoints

```
POST /api/auth/validate
Description: Validate Clarity session and get middleware JWT token
Authentication: None (uses Clarity session)

Request:
{
  "claritySessionId": "ABC123XYZ",
  "userId": "user123"
}

Response (200):
{
  "success": true,
  "middlewareToken": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "user123",
    "name": "John Doe",
    "email": "john@company.com",
    "groups": ["PM", "Developer"]
  },
  "expiresIn": 1800
}

Response (401):
{
  "success": false,
  "error": "INVALID_SESSION",
  "message": "Session validation failed with Clarity"
}

─────────────────────────────────────────────────────

POST /api/auth/refresh
Description: Refresh expired JWT token
Authentication: Bearer token (even if expired)

Request:
{
  "claritySessionId": "ABC123XYZ"
}

Response (200):
{
  "success": true,
  "middlewareToken": "eyJhbGci...",
  "expiresIn": 1800
}

Response (401):
{
  "success": false,
  "error": "SESSION_EXPIRED",
  "message": "Please log in to Clarity again"
}
```

#### Chat Endpoints

```
POST /api/chat/message
Description: Send a message to the AI assistant (non-streaming)
Authentication: Required

Request:
{
  "message": "Show me all active projects",
  "conversationId": "uuid-123" // optional
}

Response (200):
{
  "conversationId": "uuid-123",
  "messageId": "uuid-456",
  "response": "Here are your active projects:\n\n1. Project Tokyo...",
  "toolsUsed": [
    {
      "name": "clarity_list_projects",
      "input": { "status": "active" },
      "result": { "projects": [...] },
      "duration_ms": 450
    }
  ],
  "tokensUsed": 1250,
  "timestamp": "2025-11-06T10:30:45.123Z"
}

Response (400):
{
  "error": "INVALID_REQUEST",
  "message": "Message cannot be empty"
}

Response (429):
{
  "error": "RATE_LIMIT_EXCEEDED",
  "message": "You have exceeded the rate limit. Please try again in 60 seconds.",
  "retryAfter": 60
}

─────────────────────────────────────────────────────

GET /api/chat/stream
Description: Send message with streaming response (SSE)
Authentication: Required
Query Parameters:
  - token: JWT token (since SSE doesn't support headers well)

POST body:
{
  "message": "Show me tasks due this week",
  "conversationId": "uuid-123"
}

Response (200):
Content-Type: text/event-stream

data: {"type":"token","content":"Here"}
data: {"type":"token","content":" are"}
data: {"type":"token","content":" the"}
data: {"type":"token","content":" tasks"}
data: {"type":"tool_use","tool":"clarity_search_tasks","input":{...}}
data: {"type":"tool_result","result":{...}}
data: {"type":"token","content":" due"}
data: {"type":"token","content":" this"}
data: {"type":"token","content":" week:\n\n"}
data: {"type":"done"}

─────────────────────────────────────────────────────

GET /api/chat/conversations
Description: List user's conversations
Authentication: Required
Query Parameters:
  - limit: number (default: 20, max: 100)
  - offset: number (default: 0)
  - search: string (optional search term)

Response (200):
{
  "conversations": [
    {
      "id": "uuid-123",
      "title": "Project status discussion",
      "messageCount": 12,
      "createdAt": "2025-11-01T10:00:00Z",
      "updatedAt": "2025-11-06T10:30:45Z",
      "lastMessage": "Thank you! That's very helpful."
    },
    ...
  ],
  "total": 45,
  "limit": 20,
  "offset": 0
}

─────────────────────────────────────────────────────

GET /api/chat/conversations/:id
Description: Get conversation history
Authentication: Required

Response (200):
{
  "id": "uuid-123",
  "title": "Project status discussion",
  "createdAt": "2025-11-01T10:00:00Z",
  "updatedAt": "2025-11-06T10:30:45Z",
  "messageCount": 12,
  "messages": [
    {
      "id": "msg-1",
      "role": "user",
      "content": "Show me Project Tokyo status",
      "createdAt": "2025-11-01T10:00:00Z"
    },
    {
      "id": "msg-2",
      "role": "assistant",
      "content": "Project Tokyo is currently...",
      "toolCalls": [
        {
          "name": "clarity_get_project",
          "input": { "projectId": "tokyo-2025" }
        }
      ],
      "createdAt": "2025-11-01T10:00:05Z"
    },
    ...
  ]
}

Response (404):
{
  "error": "NOT_FOUND",
  "message": "Conversation not found"
}

Response (403):
{
  "error": "FORBIDDEN",
  "message": "You don't have access to this conversation"
}

─────────────────────────────────────────────────────

DELETE /api/chat/conversations/:id
Description: Delete a conversation
Authentication: Required

Response (204): No content

Response (404):
{
  "error": "NOT_FOUND",
  "message": "Conversation not found"
}

─────────────────────────────────────────────────────

PUT /api/chat/conversations/:id/title
Description: Update conversation title
Authentication: Required

Request:
{
  "title": "Project Tokyo Status Updates"
}

Response (200):
{
  "id": "uuid-123",
  "title": "Project Tokyo Status Updates",
  "updatedAt": "2025-11-06T10:30:45Z"
}
```

#### Health & Monitoring

```
GET /api/health
Description: Health check endpoint
Authentication: None

Response (200):
{
  "status": "healthy",
  "timestamp": "2025-11-06T10:30:45.123Z",
  "services": {
    "database": "connected",
    "redis": "connected",
    "clarityApi": "reachable",
    "claudeApi": "reachable"
  },
  "version": "1.0.0"
}

Response (503):
{
  "status": "unhealthy",
  "timestamp": "2025-11-06T10:30:45.123Z",
  "services": {
    "database": "connected",
    "redis": "disconnected",
    "clarityApi": "reachable",
    "claudeApi": "error"
  },
  "version": "1.0.0"
}

─────────────────────────────────────────────────────

GET /api/metrics
Description: Prometheus-style metrics
Authentication: API key (for monitoring tools)

Response (200):
# HELP api_requests_total Total number of API requests
# TYPE api_requests_total counter
api_requests_total{endpoint="/api/chat/message",status="200"} 1523
api_requests_total{endpoint="/api/chat/message",status="400"} 45
api_requests_total{endpoint="/api/chat/message",status="500"} 3

# HELP api_request_duration_seconds API request duration
# TYPE api_request_duration_seconds histogram
api_request_duration_seconds_bucket{endpoint="/api/chat/message",le="0.5"} 1200
api_request_duration_seconds_bucket{endpoint="/api/chat/message",le="1"} 1450
api_request_duration_seconds_bucket{endpoint="/api/chat/message",le="5"} 1560

# HELP ai_tokens_used_total Total AI tokens used
# TYPE ai_tokens_used_total counter
ai_tokens_used_total{user="user123"} 45000
ai_tokens_used_total{user="user456"} 32000
```

---

## Security Architecture

### Security Layers

```
┌────────────────────────────────────────────────────────┐
│  Layer 1: Network Security                             │
│  ─────────────────────────                             │
│  ✓ VPC with private subnets                            │
│  ✓ Security groups (firewall rules)                    │
│  ✓ Network ACLs                                        │
│  ✓ HTTPS only (TLS 1.3)                                │
│  ✓ AWS WAF for DDoS protection                         │
└────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────┐
│  Layer 2: Application Gateway Security                 │
│  ────────────────────────────────────                  │
│  ✓ Rate limiting (per user + global)                   │
│  ✓ IP whitelist (optional)                             │
│  ✓ Request size limits                                 │
│  ✓ SQL injection protection                            │
│  ✓ XSS protection                                      │
│  ✓ CSRF tokens                                         │
└────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────┐
│  Layer 3: Authentication & Authorization               │
│  ──────────────────────────────────────                │
│  ✓ Clarity session validation                          │
│  ✓ JWT token verification                              │
│  ✓ Token expiration (30 min)                           │
│  ✓ No token in logs or URLs                            │
│  ✓ Session fixation prevention                         │
└────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────┐
│  Layer 4: Permission Validation                        │
│  ─────────────────────────                             │
│  ✓ Check before EVERY operation                        │
│  ✓ Global rights validation                            │
│  ✓ OBS access validation                               │
│  ✓ Instance-level permissions                          │
│  ✓ No permission bypass possible                       │
└────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────┐
│  Layer 5: Data Security                                │
│  ─────────────────                                     │
│  ✓ Encryption at rest (RDS, Redis, EBS)                │
│  ✓ Encryption in transit (TLS everywhere)              │
│  ✓ Secrets in AWS Secrets Manager                      │
│  ✓ No sensitive data in logs                           │
│  ✓ PII sanitization                                    │
└────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────┐
│  Layer 6: Audit & Monitoring                           │
│  ──────────────────────                                │
│  ✓ Log all operations                                  │
│  ✓ Alert on suspicious activity                        │
│  ✓ Real-time security monitoring                       │
│  ✓ Regular security scans                              │
│  ✓ Compliance reporting                                │
└────────────────────────────────────────────────────────┘
```

### Security Configuration

```typescript
// Express security middleware setup

import helmet from 'helmet';
import cors from 'cors';
import rateLimit from 'express-rate-limit';
import mongoSanitize from 'express-mongo-sanitize';
import hpp from 'hpp';

// 1. Helmet - Security headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"], // For Clarity UI
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"],
    },
  },
  hsts: {
    maxAge: 31536000, // 1 year
    includeSubDomains: true,
    preload: true
  },
  referrerPolicy: {
    policy: 'same-origin'
  },
  xssFilter: true,
  noSniff: true,
  ieNoOpen: true
}));

// 2. CORS - Restrict to Clarity origin only
app.use(cors({
  origin: [
    'https://clarity.yourcompany.com',
    process.env.CLARITY_URL
  ],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Request-ID'],
  maxAge: 600 // 10 minutes
}));

// 3. Rate limiting
const userRateLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute per user
  keyGenerator: (req) => {
    // Use user ID from JWT
    return req.user?.id || req.ip;
  },
  message: {
    error: 'RATE_LIMIT_EXCEEDED',
    message: 'Too many requests. Please try again later.',
    retryAfter: 60
  },
  standardHeaders: true,
  legacyHeaders: false
});

const globalRateLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 1000, // 1000 requests per minute globally
  message: {
    error: 'GLOBAL_RATE_LIMIT',
    message: 'System is experiencing high load. Please try again.'
  }
});

app.use('/api/', globalRateLimiter);
app.use('/api/chat/', userRateLimiter);

// 4. Input sanitization
app.use(mongoSanitize()); // Prevent NoSQL injection
app.use(hpp()); // Prevent HTTP Parameter Pollution

// 5. Request size limits
app.use(express.json({ limit: '100kb' }));
app.use(express.urlencoded({ extended: true, limit: '100kb' }));

// 6. Custom security middleware
app.use((req, res, next) => {
  // Remove sensitive headers
  delete req.headers['x-powered-by'];
  
  // Add security headers
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  
  // Generate request ID for tracking
  req.id = crypto.randomUUID();
  res.setHeader('X-Request-ID', req.id);
  
  next();
});

// 7. JWT verification middleware
const verifyJWT = (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({
        error: 'UNAUTHORIZED',
        message: 'No token provided'
      });
    }
    
    const token = authHeader.substring(7);
    
    // Verify token
    const decoded = jwt.verify(token, JWT_SECRET, {
      issuer: 'clarity-chatbot-middleware',
      audience: 'clarity-chatbot'
    });
    
    // Check expiration
    if (decoded.exp && Date.now() >= decoded.exp * 1000) {
      return res.status(401).json({
        error: 'TOKEN_EXPIRED',
        message: 'Token has expired'
      });
    }
    
    // Add user to request
    req.user = {
      id: decoded.userId,
      name: decoded.userName,
      groups: decoded.groups
    };
    
    next();
    
  } catch (error) {
    logger.error('JWT verification failed', { error });
    return res.status(401).json({
      error: 'INVALID_TOKEN',
      message: 'Token verification failed'
    });
  }
};

// Apply JWT verification to protected routes
app.use('/api/chat/*', verifyJWT);
app.use('/api/admin/*', verifyJWT);

// 8. Audit logging middleware
app.use((req, res, next) => {
  const start = Date.now();
  
  // Log request
  logger.info('API request', {
    requestId: req.id,
    method: req.method,
    path: req.path,
    userId: req.user?.id,
    ip: req.ip,
    userAgent: req.headers['user-agent']
  });
  
  // Capture response
  const oldSend = res.send;
  res.send = function(data) {
    const duration = Date.now() - start;
    
    // Log response
    logger.info('API response', {
      requestId: req.id,
      statusCode: res.statusCode,
      duration,
      userId: req.user?.id
    });
    
    // Save to audit log if needed
    if (req.method !== 'GET' && req.user) {
      saveAuditLog({
        requestId: req.id,
        userId: req.user.id,
        action: `${req.method} ${req.path}`,
        success: res.statusCode < 400,
        duration,
        details: {
          statusCode: res.statusCode,
          path: req.path
        }
      });
    }
    
    oldSend.call(this, data);
  };
  
  next();
});
```

### Secrets Management

```typescript
// AWS Secrets Manager integration

import { SecretsManagerClient, GetSecretValueCommand } from "@aws-sdk/client-secrets-manager";

const secretsClient = new SecretsManagerClient({ region: AWS_REGION });

async function getSecret(secretName: string): Promise<any> {
  try {
    const response = await secretsClient.send(
      new GetSecretValueCommand({
        SecretId: secretName
      })
    );
    
    if (response.SecretString) {
      return JSON.parse(response.SecretString);
    }
    
    throw new Error('Secret not found');
    
  } catch (error) {
    logger.error('Failed to retrieve secret', { secretName, error });
    throw error;
  }
}

// Load secrets at startup
async function initializeSecrets() {
  const secrets = await getSecret('clarity-chatbot/production');
  
  process.env.CLARITY_API_TOKEN = secrets.CLARITY_API_TOKEN;
  process.env.CLAUDE_API_KEY = secrets.CLAUDE_API_KEY;
  process.env.JWT_SECRET = secrets.JWT_SECRET;
  process.env.DATABASE_PASSWORD = secrets.DATABASE_PASSWORD;
  process.env.REDIS_PASSWORD = secrets.REDIS_PASSWORD;
  
  logger.info('Secrets loaded successfully');
}

// Secrets stored in AWS Secrets Manager:
// {
//   "CLARITY_API_TOKEN": "...",
//   "CLAUDE_API_KEY": "sk-ant-...",
//   "JWT_SECRET": "...",
//   "DATABASE_PASSWORD": "...",
//   "REDIS_PASSWORD": "..."
// }
```

---

## Caching Strategy

### Redis Cache Architecture

```
┌────────────────────────────────────────────────────┐
│              Redis Cache Layers                    │
└────────────────────────────────────────────────────┘

Layer 1: Session Validation Cache
─────────────────────────────────
Key Pattern: session:{claritySessionId}
TTL: 5 minutes
Purpose: Avoid repeated Clarity API calls for session validation

Data Structure:
{
  "userId": "user123",
  "userName": "John Doe",
  "email": "john@company.com",
  "groups": ["PM", "Developer"],
  "validatedAt": 1699268645123,
  "expiresAt": 1699269945123
}

─────────────────────────────────────────────────────

Layer 2: User Permissions Cache
─────────────────────────────────
Key Pattern: permissions:{userId}
TTL: 15 minutes
Purpose: Cache user rights to avoid frequent permission lookups

Data Structure:
{
  "userId": "user123",
  "globalRights": [
    { "name": "project.task.edit", "granted": true },
    { "name": "project.financial.view", "granted": false }
  ],
  "groups": [
    { "id": "grp1", "name": "Project Managers", "rights": [...] }
  ],
  "obsAccess": [
    { "obsType": "department", "obsId": "eng", "accessLevel": "write" }
  ],
  "cachedAt": 1699268645123
}

─────────────────────────────────────────────────────

Layer 3: Project Data Cache
────────────────────────────
Key Pattern: project:{projectId}
TTL: 2 minutes
Purpose: Reduce Clarity API calls for frequently accessed projects

Data Structure:
{
  "id": "tokyo-2025",
  "name": "Project Tokyo",
  "status": "active",
  "startDate": "2025-01-01",
  "finishDate": "2025-12-31",
  "percentComplete": 35,
  ... (full project data)
}

─────────────────────────────────────────────────────

Layer 4: Task Data Cache
─────────────────────────
Key Pattern: task:{taskId}
TTL: 1 minute (shorter because tasks change frequently)
Purpose: Cache task details for quick access

─────────────────────────────────────────────────────

Layer 5: Project List Cache
────────────────────────────
Key Pattern: projects:list:{userId}:{filter hash}
TTL: 3 minutes
Purpose: Cache filtered project lists per user

Example:
projects:list:user123:sha256(status=active&startDate>2025-01-01)

─────────────────────────────────────────────────────

Layer 6: Rate Limit Counters
─────────────────────────────
Key Pattern: ratelimit:{userId}:{window}
TTL: 1 minute
Purpose: Track request counts for rate limiting

Data Structure: Simple counter
INCR ratelimit:user123:1699268640000
EXPIRE ratelimit:user123:1699268640000 60
```

### Cache Implementation

```typescript
// Redis cache service

import Redis from 'ioredis';

class CacheService {
  private redis: Redis;
  
  constructor() {
    this.redis = new Redis({
      host: REDIS_HOST,
      port: REDIS_PORT,
      password: REDIS_PASSWORD,
      db: 0,
      retryStrategy: (times) => {
        const delay = Math.min(times * 50, 2000);
        return delay;
      },
      enableReadyCheck: true,
      maxRetriesPerRequest: 3
    });
    
    this.redis.on('error', (error) => {
      logger.error('Redis error', { error });
    });
    
    this.redis.on('connect', () => {
      logger.info('Redis connected');
    });
  }
  
  // Generic cache methods
  async get<T>(key: string): Promise<T | null> {
    try {
      const value = await this.redis.get(key);
      if (!value) return null;
      return JSON.parse(value) as T;
    } catch (error) {
      logger.error('Cache get error', { key, error });
      return null;
    }
  }
  
  async set(key: string, value: any, ttlSeconds: number): Promise<void> {
    try {
      await this.redis.setex(
        key,
        ttlSeconds,
        JSON.stringify(value)
      );
    } catch (error) {
      logger.error('Cache set error', { key, error });
    }
  }
  
  async delete(key: string): Promise<void> {
    try {
      await this.redis.del(key);
    } catch (error) {
      logger.error('Cache delete error', { key, error });
    }
  }
  
  async deletePattern(pattern: string): Promise<number> {
    try {
      const keys = await this.redis.keys(pattern);
      if (keys.length === 0) return 0;
      return await this.redis.del(...keys);
    } catch (error) {
      logger.error('Cache delete pattern error', { pattern, error });
      return 0;
    }
  }
  
  // Session cache
  async getSession(sessionId: string): Promise<SessionData | null> {
    return this.get<SessionData>(`session:${sessionId}`);
  }
  
  async setSession(sessionId: string, data: SessionData): Promise<void> {
    await this.set(`session:${sessionId}`, data, 300); // 5 minutes
  }
  
  // Permissions cache
  async getPermissions(userId: string): Promise<UserPermissions | null> {
    return this.get<UserPermissions>(`permissions:${userId}`);
  }
  
  async setPermissions(userId: string, permissions: UserPermissions): Promise<void> {
    await this.set(`permissions:${userId}`, permissions, 900); // 15 minutes
  }
  
  async invalidateUserPermissions(userId: string): Promise<void> {
    await this.delete(`permissions:${userId}`);
  }
  
  // Project cache
  async getProject(projectId: string): Promise<ClarityProject | null> {
    return this.get<ClarityProject>(`project:${projectId}`);
  }
  
  async setProject(projectId: string, project: ClarityProject): Promise<void> {
    await this.set(`project:${projectId}`, project, 120); // 2 minutes
  }
  
  async invalidateProject(projectId: string): Promise<void> {
    await this.delete(`project:${projectId}`);
    // Also invalidate project lists that might contain this project
    await this.deletePattern('projects:list:*');
  }
  
  // Task cache
  async getTask(taskId: string): Promise<ClarityTask | null> {
    return this.get<ClarityTask>(`task:${taskId}`);
  }
  
  async setTask(taskId: string, task: ClarityTask): Promise<void> {
    await this.set(`task:${taskId}`, task, 60); // 1 minute
  }
  
  async invalidateTask(taskId: string): Promise<void> {
    await this.delete(`task:${taskId}`);
  }
  
  // Project list cache
  async getProjectList(userId: string, filterHash: string): Promise<ClarityProject[] | null> {
    return this.get<ClarityProject[]>(`projects:list:${userId}:${filterHash}`);
  }
  
  async setProjectList(userId: string, filterHash: string, projects: ClarityProject[]): Promise<void> {
    await this.set(`projects:list:${userId}:${filterHash}`, projects, 180); // 3 minutes
  }
  
  // Rate limiting
  async incrementRateLimit(userId: string): Promise<number> {
    const window = Math.floor(Date.now() / 60000) * 60000; // Current minute
    const key = `ratelimit:${userId}:${window}`;
    
    const count = await this.redis.incr(key);
    
    // Set expiry only on first increment
    if (count === 1) {
      await this.redis.expire(key, 60);
    }
    
    return count;
  }
  
  async getRateLimit(userId: string): Promise<number> {
    const window = Math.floor(Date.now() / 60000) * 60000;
    const key = `ratelimit:${userId}:${window}`;
    const count = await this.redis.get(key);
    return count ? parseInt(count, 10) : 0;
  }
  
  // Health check
  async ping(): Promise<boolean> {
    try {
      const result = await this.redis.ping();
      return result === 'PONG';
    } catch {
      return false;
    }
  }
}

export const cacheService = new CacheService();
```

### Cache Invalidation Strategy

```typescript
// When to invalidate cache

class CacheInvalidationService {
  
  // Invalidate after data modification
  async onProjectUpdated(projectId: string) {
    await cacheService.invalidateProject(projectId);
    // Also invalidate lists that might contain this project
    await cacheService.deletePattern('projects:list:*');
    
    logger.info('Cache invalidated', {
      type: 'project',
      projectId,
      reason: 'data_updated'
    });
  }
  
  async onTaskUpdated(taskId: string, projectId: string) {
    await cacheService.invalidateTask(taskId);
    // Project might have changed due to task update (progress, etc.)
    await cacheService.invalidateProject(projectId);
    
    logger.info('Cache invalidated', {
      type: 'task',
      taskId,
      projectId,
      reason: 'data_updated'
    });
  }
  
  async onUserPermissionsChanged(userId: string) {
    await cacheService.invalidateUserPermissions(userId);
    // User might now have access to different projects
    await cacheService.deletePattern(`projects:list:${userId}:*`);
    
    logger.info('Cache invalidated', {
      type: 'permissions',
      userId,
      reason: 'permissions_changed'
    });
  }
  
  // Scheduled cache warming (optional)
  async warmCache() {
    // Pre-load frequently accessed data during off-peak hours
    logger.info('Starting cache warming');
    
    // Example: Pre-load active projects for all users
    const activeUsers = await getActiveUsers();
    
    for (const user of activeUsers) {
      try {
        const projects = await clarityClient.listProjects({
          userId: user.id,
          status: 'active'
        });
        
        const filterHash = hashFilters({ status: 'active' });
        await cacheService.setProjectList(user.id, filterHash, projects);
        
      } catch (error) {
        logger.error('Cache warming failed for user', { userId: user.id, error });
      }
    }
    
    logger.info('Cache warming completed');
  }
}

// Helper: Generate hash for filter parameters
function hashFilters(filters: any): string {
  const crypto = require('crypto');
  const json = JSON.stringify(filters, Object.keys(filters).sort());
  return crypto.createHash('sha256').update(json).digest('hex').substring(0, 16);
}
```

---

## Error Handling Architecture

### Error Types & Codes

```typescript
// Standardized error codes

enum ErrorCode {
  // Authentication errors (401)
  UNAUTHORIZED = 'UNAUTHORIZED',
  INVALID_SESSION = 'INVALID_SESSION',
  SESSION_EXPIRED = 'SESSION_EXPIRED',
  TOKEN_EXPIRED = 'TOKEN_EXPIRED',
  INVALID_TOKEN = 'INVALID_TOKEN',
  
  // Authorization errors (403)
  FORBIDDEN = 'FORBIDDEN',
  PERMISSION_DENIED = 'PERMISSION_DENIED',
  INSUFFICIENT_RIGHTS = 'INSUFFICIENT_RIGHTS',
  
  // Validation errors (400)
  INVALID_REQUEST = 'INVALID_REQUEST',
  INVALID_INPUT = 'INVALID_INPUT',
  MISSING_PARAMETER = 'MISSING_PARAMETER',
  VALIDATION_FAILED = 'VALIDATION_FAILED',
  
  // Resource errors (404)
  NOT_FOUND = 'NOT_FOUND',
  PROJECT_NOT_FOUND = 'PROJECT_NOT_FOUND',
  TASK_NOT_FOUND = 'TASK_NOT_FOUND',
  CONVERSATION_NOT_FOUND = 'CONVERSATION_NOT_FOUND',
  
  // Rate limiting (429)
  RATE_LIMIT_EXCEEDED = 'RATE_LIMIT_EXCEEDED',
  GLOBAL_RATE_LIMIT = 'GLOBAL_RATE_LIMIT',
  QUOTA_EXCEEDED = 'QUOTA_EXCEEDED',
  
  // External service errors (502, 503)
  CLARITY_API_ERROR = 'CLARITY_API_ERROR',
  CLARITY_API_TIMEOUT = 'CLARITY_API_TIMEOUT',
  CLARITY_API_UNAVAILABLE = 'CLARITY_API_UNAVAILABLE',
  AI_API_ERROR = 'AI_API_ERROR',
  AI_API_TIMEOUT = 'AI_API_TIMEOUT',
  AI_API_UNAVAILABLE = 'AI_API_UNAVAILABLE',
  DATABASE_ERROR = 'DATABASE_ERROR',
  CACHE_ERROR = 'CACHE_ERROR',
  
  // Business logic errors (400, 409)
  OPERATION_FAILED = 'OPERATION_FAILED',
  CONFLICT = 'CONFLICT',
  CIRCULAR_DEPENDENCY = 'CIRCULAR_DEPENDENCY',
  INVALID_STATE_TRANSITION = 'INVALID_STATE_TRANSITION',
  
  // Server errors (500)
  INTERNAL_ERROR = 'INTERNAL_ERROR',
  UNHANDLED_ERROR = 'UNHANDLED_ERROR'
}

// Custom error class
class AppError extends Error {
  public code: ErrorCode;
  public statusCode: number;
  public isOperational: boolean;
  public details?: any;
  
  constructor(
    message: string,
    code: ErrorCode,
    statusCode: number,
    isOperational: boolean = true,
    details?: any
  ) {
    super(message);
    this.name = this.constructor.name;
    this.code = code;
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    this.details = details;
    
    Error.captureStackTrace(this, this.constructor);
  }
}

// Specific error types
class AuthenticationError extends AppError {
  constructor(message: string, code: ErrorCode = ErrorCode.UNAUTHORIZED, details?: any) {
    super(message, code, 401, true, details);
  }
}

class AuthorizationError extends AppError {
  constructor(message: string, code: ErrorCode = ErrorCode.PERMISSION_DENIED, details?: any) {
    super(message, code, 403, true, details);
  }
}

class ValidationError extends AppError {
  constructor(message: string, details?: any) {
    super(message, ErrorCode.VALIDATION_FAILED, 400, true, details);
  }
}

class NotFoundError extends AppError {
  constructor(message: string, code: ErrorCode = ErrorCode.NOT_FOUND, details?: any) {
    super(message, code, 404, true, details);
  }
}

class RateLimitError extends AppError {
  constructor(message: string, retryAfter: number) {
    super(message, ErrorCode.RATE_LIMIT_EXCEEDED, 429, true, { retryAfter });
  }
}

class ExternalServiceError extends AppError {
  constructor(
    service: string,
    message: string,
    code: ErrorCode,
    statusCode: number = 502,
    details?: any
  ) {
    super(message, code, statusCode, true, { ...details, service });
  }
}
```

### Error Handling Middleware

```typescript
// Global error handler

function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  // Log the error
  logger.error('Error occurred', {
    requestId: req.id,
    error: {
      name: err.name,
      message: err.message,
      stack: err.stack
    },
    request: {
      method: req.method,
      path: req.path,
      userId: req.user?.id,
      ip: req.ip
    }
  });
  
  // Handle known error types
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: err.code,
      message: err.message,
      details: err.details,
      requestId: req.id
    });
  }
  
  // Handle Joi validation errors
  if (err.name === 'ValidationError') {
    return res.status(400).json({
      error: ErrorCode.VALIDATION_FAILED,
      message: err.message,
      requestId: req.id
    });
  }
  
  // Handle JWT errors
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({
      error: ErrorCode.INVALID_TOKEN,
      message: 'Invalid authentication token',
      requestId: req.id
    });
  }
  
  if (err.name === 'TokenExpiredError') {
    return res.status(401).json({
      error: ErrorCode.TOKEN_EXPIRED,
      message: 'Authentication token has expired',
      requestId: req.id
    });
  }
  
  // Handle database errors
  if (err.name === 'SequelizeConnectionError' || err.name === 'MongoNetworkError') {
    return res.status(503).json({
      error: ErrorCode.DATABASE_ERROR,
      message: 'Database temporarily unavailable',
      requestId: req.id
    });
  }
  
  // Handle Axios errors (external API calls)
  if (err.isAxiosError) {
    const statusCode = err.response?.status || 502;
    const errorData = err.response?.data;
    
    // Determine which service failed
    const service = err.config?.baseURL?.includes('clarity') ? 'Clarity API' : 'External API';
    
    return res.status(statusCode).json({
      error: service === 'Clarity API' ? ErrorCode.CLARITY_API_ERROR : ErrorCode.AI_API_ERROR,
      message: `${service} request failed`,
      details: {
        statusCode,
        message: errorData?.message || err.message
      },
      requestId: req.id
    });
  }
  
  // Unhandled errors - don't expose details to client
  logger.error('Unhandled error', {
    requestId: req.id,
    error: err,
    stack: err.stack
  });
  
  res.status(500).json({
    error: ErrorCode.INTERNAL_ERROR,
    message: 'An unexpected error occurred',
    requestId: req.id
  });
}

app.use(errorHandler);
```

### Graceful Degradation

```typescript
// Handle service unavailability gracefully

class ServiceResilienceManager {
  private circuitBreakers: Map<string, CircuitBreaker> = new Map();
  
  // Circuit breaker pattern
  async callWithCircuitBreaker<T>(
    serviceName: string,
    fn: () => Promise<T>,
    fallback?: () => Promise<T>
  ): Promise<T> {
    
    let breaker = this.circuitBreakers.get(serviceName);
    
    if (!breaker) {
      breaker = new CircuitBreaker({
        failureThreshold: 5,
        resetTimeout: 60000, // 1 minute
        monitoringPeriod: 10000 // 10 seconds
      });
      this.circuitBreakers.set(serviceName, breaker);
    }
    
    if (breaker.isOpen()) {
      logger.warn('Circuit breaker open', { service: serviceName });
      
      if (fallback) {
        return fallback();
      }
      
      throw new ExternalServiceError(
        serviceName,
        `${serviceName} is temporarily unavailable`,
        ErrorCode.SERVICE_UNAVAILABLE,
        503
      );
    }
    
    try {
      const result = await fn();
      breaker.recordSuccess();
      return result;
      
    } catch (error) {
      breaker.recordFailure();
      
      if (fallback && breaker.isOpen()) {
        logger.info('Using fallback', { service: serviceName });
        return fallback();
      }
      
      throw error;
    }
  }
  
  // Retry with exponential backoff
  async retryWithBackoff<T>(
    fn: () => Promise<T>,
    maxRetries: number = 3,
    baseDelay: number = 1000
  ): Promise<T> {
    
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        return await fn();
        
      } catch (error) {
        if (attempt === maxRetries) {
          throw error;
        }
        
        // Check if error is retryable
        if (!this.isRetryableError(error)) {
          throw error;
        }
        
        const delay = baseDelay * Math.pow(2, attempt);
        logger.info('Retrying request', {
          attempt: attempt + 1,
          maxRetries,
          delay,
          error: error.message
        });
        
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
    
    throw new Error('Max retries exceeded');
  }
  
  private isRetryableError(error: any): boolean {
    // Network errors are retryable
    if (error.code === 'ECONNREFUSED' || error.code === 'ETIMEDOUT') {
      return true;
    }
    
    // Some HTTP status codes are retryable
    if (error.response) {
      const status = error.response.status;
      return status === 408 || status === 429 || status >= 500;
    }
    
    return false;
  }
}

// Circuit breaker implementation
class CircuitBreaker {
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  private failureCount: number = 0;
  private successCount: number = 0;
  private lastFailureTime: number | null = null;
  
  constructor(private config: {
    failureThreshold: number;
    resetTimeout: number;
    monitoringPeriod: number;
  }) {}
  
  isOpen(): boolean {
    if (this.state === 'open') {
      // Check if we should try half-open
      if (this.lastFailureTime && 
          Date.now() - this.lastFailureTime >= this.config.resetTimeout) {
        this.state = 'half-open';
        this.failureCount = 0;
        return false;
      }
      return true;
    }
    return false;
  }
  
  recordSuccess() {
    this.failureCount = 0;
    
    if (this.state === 'half-open') {
      this.state = 'closed';
      logger.info('Circuit breaker closed');
    }
  }
  
  recordFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.config.failureThreshold) {
      this.state = 'open';
      logger.warn('Circuit breaker opened', {
        failures: this.failureCount,
        threshold: this.config.failureThreshold
      });
    }
  }
}

export const resilienceManager = new ServiceResilienceManager();
```

---

## Monitoring & Observability

### CloudWatch Dashboard

```typescript
// Metrics to track

const METRICS = {
  // API metrics
  'API.Requests.Total': 'Count',
  'API.Requests.Success': 'Count',
  'API.Requests.Error': 'Count',
  'API.Response.Time': 'Milliseconds',
  'API.Response.Time.P95': 'Milliseconds',
  'API.Response.Time.P99': 'Milliseconds',
  
  // Chat metrics
  'Chat.Messages.Sent': 'Count',
  'Chat.Messages.Received': 'Count',
  'Chat.Conversations.Active': 'Count',
  'Chat.Streaming.Duration': 'Milliseconds',
  
  // MCP metrics
  'MCP.Tools.Executed': 'Count',
  'MCP.Tools.Success': 'Count',
  'MCP.Tools.Error': 'Count',
  'MCP.Tools.Duration': 'Milliseconds',
  
  // Permission checks
  'Permissions.Checks.Total': 'Count',
  'Permissions.Checks.Allowed': 'Count',
  'Permissions.Checks.Denied': 'Count',
  'Permissions.Checks.Duration': 'Milliseconds',
  
  // External services
  'Clarity.API.Requests': 'Count',
  'Clarity.API.Errors': 'Count',
  'Clarity.API.Duration': 'Milliseconds',
  'AI.API.Requests': 'Count',
  'AI.API.Tokens.Input': 'Count',
  'AI.API.Tokens.Output': 'Count',
  'AI.API.Cost': 'None', // USD
  
  // Cache metrics
  'Cache.Hits': 'Count',
  'Cache.Misses': 'Count',
  'Cache.Hit.Rate': 'Percent',
  
  // Rate limiting
  'RateLimit.Hits': 'Count',
  'RateLimit.Blocks': 'Count',
  
  // System metrics
  'System.CPU.Usage': 'Percent',
  'System.Memory.Usage': 'Percent',
  'System.Disk.Usage': 'Percent',
  'Database.Connections.Active': 'Count'
};

// CloudWatch metrics publisher
import { CloudWatchClient, PutMetricDataCommand } from "@aws-sdk/client-cloudwatch";

class MetricsPublisher {
  private cloudwatch: CloudWatchClient;
  private namespace: string = 'ClarityChatbot';
  
  constructor() {
    this.cloudwatch = new CloudWatchClient({ region: AWS_REGION });
  }
  
  async publish(metricName: string, value: number, unit: string = 'None', dimensions?: Record<string, string>) {
    const metricDimensions = Object.entries(dimensions || {}).map(([name, value]) => ({
      Name: name,
      Value: value
    }));
    
    try {
      await this.cloudwatch.send(new PutMetricDataCommand({
        Namespace: this.namespace,
        MetricData: [
          {
            MetricName: metricName,
            Value: value,
            Unit: unit,
            Timestamp: new Date(),
            Dimensions: metricDimensions
          }
        ]
      }));
    } catch (error) {
      logger.error('Failed to publish metric', { metricName, error });
    }
  }
  
  async publishMultiple(metrics: Array<{name: string, value: number, unit?: string, dimensions?: Record<string, string>}>) {
    const metricData = metrics.map(m => ({
      MetricName: m.name,
      Value: m.value,
      Unit: m.unit || 'None',
      Timestamp: new Date(),
      Dimensions: Object.entries(m.dimensions || {}).map(([name, value]) => ({
        Name: name,
        Value: value
      }))
    }));
    
    try {
      await this.cloudwatch.send(new PutMetricDataCommand({
        Namespace: this.namespace,
        MetricData: metricData
      }));
    } catch (error) {
      logger.error('Failed to publish metrics', { error });
    }
  }
}

export const metricsPublisher = new MetricsPublisher();
```

### Alert Configuration

```yaml
# CloudWatch Alarms

Alarms:
  # High error rate
  - Name: HighErrorRate
    Metric: API.Requests.Error
    Statistic: Sum
    Period: 300 # 5 minutes
    EvaluationPeriods: 2
    Threshold: 50 # 50 errors in 5 min
    ComparisonOperator: GreaterThanThreshold
    Actions:
      - SendSNSNotification: ops-team@company.com
      - SendSlackAlert: #ops-alerts
    
  # High response time
  - Name: HighResponseTime
    Metric: API.Response.Time.P95
    Statistic: Average
    Period: 300
    EvaluationPeriods: 2
    Threshold: 5000 # 5 seconds
    ComparisonOperator: GreaterThanThreshold
    Actions:
      - SendSNSNotification: ops-team@company.com
  
  # High AI costs
  - Name: HighAICosts
    Metric: AI.API.Cost
    Statistic: Sum
    Period: 3600 # 1 hour
    EvaluationPeriods: 1
    Threshold: 50 # $50/hour
    ComparisonOperator: GreaterThanThreshold
    Actions:
      - SendSNSNotification: finance@company.com
      - SendSlackAlert: #cost-alerts
  
  # Clarity API errors
  - Name: ClarityAPIErrors
    Metric: Clarity.API.Errors
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 20
    ComparisonOperator: GreaterThanThreshold
    Actions:
      - SendSNSNotification: ops-team@company.com
  
  # Low cache hit rate
  - Name: LowCacheHitRate
    Metric: Cache.Hit.Rate
    Statistic: Average
    Period: 600 # 10 minutes
    EvaluationPeriods: 3
    Threshold: 50 # 50%
    ComparisonOperator: LessThanThreshold
    Actions:
      - SendSlackAlert: #ops-alerts
  
  # High memory usage
  - Name: HighMemoryUsage
    Metric: System.Memory.Usage
    Statistic: Average
    Period: 300
    EvaluationPeriods: 3
    Threshold: 85 # 85%
    ComparisonOperator: GreaterThanThreshold
    Actions:
      - SendSNSNotification: ops-team@company.com
  
  # Database connection issues
  - Name: HighDatabaseConnections
    Metric: Database.Connections.Active
    Statistic: Average
    Period: 300
    EvaluationPeriods: 2
    Threshold: 80 # 80% of pool
    ComparisonOperator: GreaterThanThreshold
    Actions:
      - SendSlackAlert: #ops-alerts
```

---

## Summary

This document provides comprehensive architecture diagrams and technical specifications for the Clarity PPM AI Chatbot system, including:

✓ Complete system architecture with 3-layer design  
✓ AWS deployment architecture with VPC, security groups, and services  
✓ Detailed authentication and session management flows  
✓ Permission validation architecture with caching  
✓ End-to-end chat message flow with MCP integration  
✓ Complete MCP tool specifications (15 tools)  
✓ Database schemas and TypeScript interfaces  
✓ REST API specifications with request/response formats  
✓ Multi-layer security architecture  
✓ Redis caching strategy with invalidation patterns  
✓ Error handling and resilience patterns  
✓ Monitoring and observability with CloudWatch

**Next Steps:**
1. Review and validate architecture with stakeholders
2. Adjust based on specific requirements
3. Begin implementation following the task breakdown
4. Set up infrastructure using IaC (Terraform/CloudFormation)
5. Implement security controls first
6. Build and test iteratively

---

**Document Version:** 1.0  
**Created:** 2025-11-06  
**Last Updated:** 2025-11-06
