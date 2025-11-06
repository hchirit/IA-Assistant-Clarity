# Clarity PPM AI Chatbot - Mermaid Diagrams
# Can be rendered in: GitHub, GitLab, draw.io, VSCode, Notion, etc.

# ============================================
# 1. SYSTEM ARCHITECTURE OVERVIEW (3 LAYERS)
# ============================================

```mermaid
graph TB
    subgraph Presentation["🖥️ PRESENTATION LAYER"]
        CP[Clarity PPM Custom Page]
        UI[Chat Interface UI/UX]
        SM[Session Manager]
        AC[API Client]
        
        CP --> UI
        CP --> SM
        UI --> AC
        SM --> AC
    end
    
    subgraph Application["⚙️ APPLICATION LAYER (AWS EC2)"]
        subgraph Middleware["Middleware Server"]
            AG[API Gateway]
            Auth[Auth Module]
            RL[Rate Limiter]
            Router[Request Router]
            
            AG --> Auth
            AG --> RL
            AG --> Router
        end
        
        subgraph Services["Business Services"]
            CS[Chat Service]
            PS[Permission Service]
            ClaS[Clarity Service]
            ConvS[Conversation Service]
            
            Router --> CS
            Router --> PS
            CS --> ClaS
            CS --> ConvS
        end
        
        subgraph MCP["MCP Server"]
            MPH[MCP Protocol Handler]
            Tools[MCP Tools 15]
            
            CS --> MPH
            MPH --> Tools
        end
        
        subgraph Cache["Redis Cache"]
            SessionC[Sessions]
            PermC[Permissions]
            DataC[Project/Task Data]
        end
        
        subgraph DB["PostgreSQL"]
            ConvDB[Conversations]
            MsgDB[Messages]
            AuditDB[Audit Logs]
        end
        
        Services --> Cache
        Services --> DB
    end
    
    subgraph Integration["🔌 INTEGRATION LAYER"]
        ClarityAPI[Clarity REST API]
        ClaudeAPI[Claude API]
        CW[CloudWatch]
    end
    
    AC -->|HTTPS + JWT| AG
    Tools --> ClarityAPI
    CS --> ClaudeAPI
    Application --> CW
    
    style Presentation fill:#e1f5ff
    style Application fill:#fff4e6
    style Integration fill:#f3e5f5
    style Middleware fill:#bbdefb
    style Services fill:#ffecb3
    style MCP fill:#c8e6c9
    style Cache fill:#ffccbc
    style DB fill:#d1c4e9
```

# ============================================
# 2. AWS DEPLOYMENT ARCHITECTURE
# ============================================

```mermaid
graph TB
    Internet[🌐 Internet]
    
    subgraph AWS["☁️ AWS Region eu-west-1"]
        subgraph VPC["VPC 10.0.0.0/16"]
            
            subgraph PublicSubnet["Public Subnet 10.0.1.0/24"]
                ALB[Application Load Balancer]
                WAF[AWS WAF]
                NAT[NAT Gateway]
                Bastion[Bastion Host]
                
                Internet --> ALB
                ALB --> WAF
            end
            
            subgraph PrivateSubnet["Private Subnet 10.0.10.0/24"]
                EC2[EC2 Linux Ubuntu<br/>t3.medium<br/>Middleware + MCP]
                Redis[(ElastiCache Redis<br/>cache.t3.micro)]
                RDS[(RDS PostgreSQL<br/>db.t3.micro)]
                
                WAF --> EC2
                EC2 --> Redis
                EC2 --> RDS
                EC2 --> NAT
            end
            
            subgraph ClarityNet["Clarity Network"]
                ClarityWin[EC2 Windows Server<br/>Clarity PPM]
            end
            
        end
        
        subgraph AWSServices["AWS Services"]
            CW[CloudWatch Logs/Metrics]
            SM[Secrets Manager]
            SSM[Systems Manager]
            ACM[Certificate Manager]
        end
        
        EC2 --> CW
        EC2 --> SM
    end
    
    NAT --> Internet
    ClarityWin -->|HTTPS| ALB
    EC2 -->|API Calls| Internet
    
    style PublicSubnet fill:#ffebee
    style PrivateSubnet fill:#e8f5e9
    style ClarityNet fill:#fff3e0
    style AWSServices fill:#e3f2fd
```

# ============================================
# 3. AUTHENTICATION & SESSION FLOW
# ============================================

```mermaid
sequenceDiagram
    participant U as User Browser
    participant CP as Clarity Page
    participant MW as Middleware
    participant Redis as Redis Cache
    participant Clarity as Clarity API
    
    U->>CP: 1. Navigate to chatbot page
    activate CP
    CP->>CP: 2. Extract JSESSIONID cookie
    CP->>CP: 3. Get user ID from Clarity
    
    CP->>MW: 4. POST /api/auth/validate<br/>{claritySessionId, userId}
    activate MW
    
    MW->>Redis: 5. Check cache for session
    alt Session in cache
        Redis-->>MW: Cached session data
        MW->>MW: 6. Generate JWT
        MW-->>CP: 7. Success + JWT token
    else Session not in cache
        MW->>Clarity: 8. Validate session
        Clarity-->>MW: 9. User data + permissions
        MW->>Redis: 10. Cache session (5 min TTL)
        MW->>MW: 11. Generate JWT (30 min exp)
        MW-->>CP: 12. Success + JWT token
    end
    deactivate MW
    
    CP->>CP: 13. Store JWT in memory
    CP-->>U: 14. Display ready UI
    deactivate CP
    
    Note over U,CP: User is authenticated
    
    U->>CP: 15. Type message
    CP->>MW: 16. POST /api/chat/message<br/>Bearer JWT
    activate MW
    MW->>MW: 17. Verify JWT
    MW->>MW: 18. Extract userId
    MW->>MW: 19. Process request
    MW-->>CP: 20. Response
    deactivate MW
```

# ============================================
# 4. PERMISSION VALIDATION FLOW
# ============================================

```mermaid
flowchart TD
    Start([MCP Tool Request]) --> Extract[Extract userId & resourceId]
    Extract --> CheckCache{Check Redis<br/>for permissions}
    
    CheckCache -->|Hit| ValidateGlobal[Validate Global Rights]
    CheckCache -->|Miss| FetchPerms[Fetch from Clarity API]
    
    FetchPerms --> CachePerms[Cache permissions<br/>15 min TTL]
    CachePerms --> ValidateGlobal
    
    ValidateGlobal -->|Has Right| ValidateOBS[Validate OBS Access]
    ValidateGlobal -->|No Right| DenyGlobal[Deny: Lacks global right]
    
    ValidateOBS -->|Has Access| ValidateInstance[Validate Instance Access]
    ValidateOBS -->|No Access| DenyOBS[Deny: Lacks OBS access]
    
    ValidateInstance -->|Has Access| Allow[✅ ALLOW: Execute Tool]
    ValidateInstance -->|No Access| DenyInstance[❌ DENY: No resource access]
    
    DenyGlobal --> LogDenial[Log denial reason]
    DenyOBS --> LogDenial
    DenyInstance --> LogDenial
    
    Allow --> ExecuteTool[Execute MCP Tool]
    ExecuteTool --> ReturnResult([Return Result])
    
    LogDenial --> ReturnError([Return Error with Reason])
    
    style Start fill:#e1f5ff
    style Allow fill:#c8e6c9
    style ExecuteTool fill:#a5d6a7
    style ReturnResult fill:#81c784
    style DenyGlobal fill:#ffcdd2
    style DenyOBS fill:#ffcdd2
    style DenyInstance fill:#ffcdd2
    style LogDenial fill:#ef9a9a
    style ReturnError fill:#e57373
```

# ============================================
# 5. CHAT MESSAGE FLOW WITH AI & MCP
# ============================================

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend
    participant MW as Middleware
    participant Claude as Claude API
    participant MCP as MCP Server
    participant Clarity as Clarity API
    
    U->>FE: 1. Type message
    FE->>MW: 2. POST /api/chat/message + JWT
    activate MW
    
    MW->>MW: 3. Verify JWT
    MW->>MW: 4. Load conversation history
    
    MW->>Claude: 5. Send message + MCP config
    activate Claude
    
    Claude->>Claude: 6. Process message
    Claude->>MCP: 7. Call MCP tool<br/>clarity_get_project
    activate MCP
    
    MCP->>MCP: 8. Validate permissions
    
    alt Permission Denied
        MCP-->>Claude: Error: Permission denied
        Claude-->>MW: Error message
        MW-->>FE: Show error
        FE-->>U: "No access to project"
    else Permission Allowed
        MCP->>Clarity: 9. GET /api/v1/projects/{id}
        Clarity-->>MCP: 10. Project data
        MCP-->>Claude: 11. Tool result
        deactivate MCP
        
        Claude->>Claude: 12. Generate response
        
        loop Streaming tokens
            Claude-->>MW: 13. Token chunk
            MW-->>FE: 14. SSE event
            FE-->>U: 15. Display in real-time
        end
        
        Claude-->>MW: Response complete
        deactivate Claude
        
        MW->>MW: 16. Save to database
        MW-->>FE: 17. Done event
        deactivate MW
    end
```

# ============================================
# 6. MCP TOOL EXECUTION DETAILED
# ============================================

```mermaid
flowchart TD
    ToolCall[Claude calls MCP tool] --> Receive[MCP Server receives request]
    Receive --> ValidateSchema[Validate input schema]
    
    ValidateSchema -->|Invalid| SchemaError[Return schema error]
    ValidateSchema -->|Valid| ExtractContext[Extract user context]
    
    ExtractContext --> PermCheck{Permission<br/>validation}
    
    PermCheck -->|Denied| PermError[Return permission error]
    PermCheck -->|Allowed| CallAPI[Call Clarity API]
    
    CallAPI -->|Success| ParseResponse[Parse API response]
    CallAPI -->|Error| APIError[Handle API error]
    
    ParseResponse --> FilterData[Filter by user permissions]
    FilterData --> FormatResult[Format result for Claude]
    FormatResult --> LogSuccess[Log successful execution]
    LogSuccess --> ReturnSuccess[Return formatted result]
    
    APIError --> LogError[Log error details]
    LogError --> ReturnError[Return error to Claude]
    
    SchemaError --> ReturnErrorResponse([Error Response])
    PermError --> ReturnErrorResponse
    ReturnError --> ReturnErrorResponse
    ReturnSuccess --> SuccessResponse([Success Response])
    
    style ToolCall fill:#e3f2fd
    style PermCheck fill:#fff9c4
    style CallAPI fill:#c5cae9
    style ReturnSuccess fill:#c8e6c9
    style PermError fill:#ffcdd2
    style APIError fill:#ffcdd2
    style SchemaError fill:#ffcdd2
```

# ============================================
# 7. REDIS CACHING STRATEGY
# ============================================

```mermaid
graph LR
    subgraph CacheLayers["Redis Cache Layers"]
        direction TB
        
        L1[Layer 1: Sessions<br/>TTL: 5 min<br/>session:sessionId]
        L2[Layer 2: Permissions<br/>TTL: 15 min<br/>permissions:userId]
        L3[Layer 3: Projects<br/>TTL: 2 min<br/>project:projectId]
        L4[Layer 4: Tasks<br/>TTL: 1 min<br/>task:taskId]
        L5[Layer 5: Project Lists<br/>TTL: 3 min<br/>projects:list:userId:hash]
        L6[Layer 6: Rate Limits<br/>TTL: 1 min<br/>ratelimit:userId:window]
    end
    
    subgraph Operations["Cache Operations"]
        Read[Read Operation]
        Write[Write/Update Operation]
        Delete[Delete Operation]
    end
    
    Read -->|Check L1| L1
    Read -->|Check L2| L2
    Read -->|Check L3| L3
    
    Write -->|Invalidate| L3
    Write -->|Invalidate| L4
    Write -->|Invalidate| L5
    
    Delete -->|Clear| L2
    Delete -->|Clear| L3
    Delete -->|Clear| L5
    
    style L1 fill:#e1f5ff
    style L2 fill:#f3e5f5
    style L3 fill:#fff9c4
    style L4 fill:#ffecb3
    style L5 fill:#c8e6c9
    style L6 fill:#ffccbc
```

# ============================================
# 8. SECURITY ARCHITECTURE (6 LAYERS)
# ============================================

```mermaid
graph TB
    subgraph Layer1["Layer 1: Network Security"]
        VPC[VPC with Private Subnets]
        SG[Security Groups]
        TLS[TLS 1.3 Encryption]
        WAF2[AWS WAF]
    end
    
    subgraph Layer2["Layer 2: Gateway Security"]
        RateLimit[Rate Limiting]
        IPWhitelist[IP Whitelist]
        SQLInject[SQL Injection Protection]
        XSS[XSS Protection]
        CSRF[CSRF Protection]
    end
    
    subgraph Layer3["Layer 3: Authentication"]
        SessionVal[Session Validation]
        JWTVerify[JWT Verification]
        TokenExp[Token Expiration]
        NoLeaks[No Tokens in Logs]
    end
    
    subgraph Layer4["Layer 4: Authorization"]
        PermCheck[Permission Validation]
        GlobalRights[Global Rights]
        OBSCheck[OBS Access]
        InstancePerm[Instance Permissions]
    end
    
    subgraph Layer5["Layer 5: Data Security"]
        EncryptRest[Encryption at Rest]
        EncryptTransit[Encryption in Transit]
        SecretsMgr[Secrets Manager]
        NoSensitive[No PII in Logs]
    end
    
    subgraph Layer6["Layer 6: Audit & Monitoring"]
        LogAll[Log All Operations]
        Alerts[Security Alerts]
        Monitoring[Real-time Monitoring]
        Compliance[Compliance Reporting]
    end
    
    Request[Incoming Request] --> Layer1
    Layer1 --> Layer2
    Layer2 --> Layer3
    Layer3 --> Layer4
    Layer4 --> Layer5
    Layer5 --> Layer6
    Layer6 --> Allowed[✅ Request Processed]
    
    Layer1 -.->|Blocked| Denied[❌ Request Denied]
    Layer2 -.->|Blocked| Denied
    Layer3 -.->|Blocked| Denied
    Layer4 -.->|Blocked| Denied
    
    style Layer1 fill:#ffebee
    style Layer2 fill:#fff3e0
    style Layer3 fill:#e8f5e9
    style Layer4 fill:#e1f5ff
    style Layer5 fill:#f3e5f5
    style Layer6 fill:#fff9c4
    style Allowed fill:#c8e6c9
    style Denied fill:#ffcdd2
```

# ============================================
# 9. ERROR HANDLING FLOW
# ============================================

```mermaid
flowchart TD
    Error[Error Occurs] --> Identify{Identify Error Type}
    
    Identify -->|Auth Error| AuthError[401/403<br/>Authentication/Authorization]
    Identify -->|Validation Error| ValError[400<br/>Invalid Input]
    Identify -->|Not Found| NotFound[404<br/>Resource Not Found]
    Identify -->|Rate Limit| RateError[429<br/>Rate Limit Exceeded]
    Identify -->|External Service| ExtError[502/503<br/>Service Unavailable]
    Identify -->|Internal Error| IntError[500<br/>Internal Server Error]
    
    AuthError --> LogAuth[Log authentication failure]
    ValError --> LogVal[Log validation error]
    NotFound --> LogNF[Log resource not found]
    RateError --> LogRate[Log rate limit hit]
    ExtError --> CheckCircuit{Circuit<br/>Breaker Open?}
    IntError --> LogInt[Log internal error]
    
    CheckCircuit -->|Yes| UseFallback[Use Fallback Response]
    CheckCircuit -->|No| Retry{Retry<br/>Operation?}
    
    Retry -->|Yes| RetryBackoff[Exponential Backoff]
    Retry -->|No| ServiceError[Return Service Error]
    
    RetryBackoff --> RetrySuccess{Success?}
    RetrySuccess -->|Yes| Success[Return Success]
    RetrySuccess -->|No| ServiceError
    
    LogAuth --> FormatError[Format Error Response]
    LogVal --> FormatError
    LogNF --> FormatError
    LogRate --> FormatError
    UseFallback --> FormatError
    ServiceError --> FormatError
    LogInt --> FormatError
    
    FormatError --> SendAlert{Severity<br/>High?}
    SendAlert -->|Yes| Alert[Send Alert to Ops]
    SendAlert -->|No| NoAlert[Continue]
    
    Alert --> Return[Return Error to User]
    NoAlert --> Return
    Success --> Return
    
    style Error fill:#ffebee
    style AuthError fill:#ffcdd2
    style ValError fill:#ffcdd2
    style NotFound fill:#ffcdd2
    style RateError fill:#ffe0b2
    style ExtError fill:#fff9c4
    style IntError fill:#d32f2f,color:#fff
    style Success fill:#c8e6c9
```

# ============================================
# 10. MONITORING DASHBOARD METRICS
# ============================================

```mermaid
graph TB
    subgraph Dashboard["CloudWatch Dashboard"]
        subgraph APIMetrics["API Metrics"]
            ReqTotal[Total Requests]
            ReqSuccess[Success Rate]
            ReqError[Error Rate]
            RespTime[Response Time P95]
        end
        
        subgraph ChatMetrics["Chat Metrics"]
            MsgSent[Messages Sent]
            ConvActive[Active Conversations]
            StreamDur[Streaming Duration]
        end
        
        subgraph MCPMetrics["MCP Metrics"]
            ToolExec[Tools Executed]
            ToolSuccess[Tool Success Rate]
            ToolDuration[Tool Duration]
        end
        
        subgraph PermMetrics["Permission Metrics"]
            PermChecks[Permission Checks]
            PermAllowed[Allowed %]
            PermDenied[Denied %]
        end
        
        subgraph ExtMetrics["External Services"]
            ClarityReq[Clarity API Requests]
            ClarityErr[Clarity Errors]
            AITokens[AI Tokens Used]
            AICost[AI Cost USD]
        end
        
        subgraph CacheMetrics["Cache Metrics"]
            CacheHit[Cache Hit Rate]
            CacheMiss[Cache Misses]
        end
        
        subgraph SysMetrics["System Metrics"]
            CPU[CPU Usage]
            Memory[Memory Usage]
            DBConn[DB Connections]
        end
        
        subgraph Alerts["Alerts Configuration"]
            Alert1[High Error Rate > 5%]
            Alert2[Response Time > 5s]
            Alert3[AI Cost > $50/hr]
            Alert4[Memory > 85%]
        end
    end
    
    Dashboard --> SNS[SNS Notifications]
    Dashboard --> Slack[Slack #ops-alerts]
    Dashboard --> Email[Email ops-team@]
    
    style APIMetrics fill:#e3f2fd
    style ChatMetrics fill:#f3e5f5
    style MCPMetrics fill:#e8f5e9
    style PermMetrics fill:#fff9c4
    style ExtMetrics fill:#ffecb3
    style CacheMetrics fill:#ffccbc
    style SysMetrics fill:#d1c4e9
    style Alerts fill:#ffcdd2
```

# ============================================
# 11. DEPLOYMENT PIPELINE
# ============================================

```mermaid
flowchart LR
    subgraph Dev["Development"]
        Code[Write Code]
        Test[Unit Tests]
        Commit[Git Commit]
    end
    
    subgraph CI["CI Pipeline GitHub Actions"]
        Build[Build]
        Lint[Lint & Format]
        UnitTest[Run Unit Tests]
        IntTest[Integration Tests]
        Security[Security Scan]
    end
    
    subgraph CD["CD Pipeline"]
        BuildImg[Build Docker Image]
        Push[Push to ECR]
        Deploy[Deploy to Staging]
        SmokeTest[Smoke Tests]
    end
    
    subgraph Prod["Production"]
        Approve[Manual Approval]
        BlueGreen[Blue/Green Deploy]
        Monitor[Monitor Metrics]
        Rollback{Issues?}
    end
    
    Code --> Test
    Test --> Commit
    Commit --> Build
    Build --> Lint
    Lint --> UnitTest
    UnitTest --> IntTest
    IntTest --> Security
    Security -->|Pass| BuildImg
    Security -->|Fail| NotifyFail[Notify Team]
    
    BuildImg --> Push
    Push --> Deploy
    Deploy --> SmokeTest
    SmokeTest -->|Pass| Approve
    SmokeTest -->|Fail| NotifyFail
    
    Approve --> BlueGreen
    BlueGreen --> Monitor
    Monitor --> Rollback
    Rollback -->|Yes| RollbackAction[Rollback to Previous]
    Rollback -->|No| Success[✅ Deployment Success]
    
    style Dev fill:#e8f5e9
    style CI fill:#e3f2fd
    style CD fill:#fff9c4
    style Prod fill:#f3e5f5
    style Success fill:#c8e6c9
    style NotifyFail fill:#ffcdd2
    style RollbackAction fill:#ffccbc
```

# ============================================
# 12. DATA FLOW DIAGRAM
# ============================================

```mermaid
flowchart TD
    User[👤 User] -->|1. Login| Clarity[Clarity PPM]
    Clarity -->|2. Navigate| ChatPage[Chat Page]
    ChatPage -->|3. Extract Session| SessionToken{Session Token}
    
    SessionToken -->|4. Validate| Middleware[Middleware API]
    Middleware -->|5. Check Cache| Redis[(Redis)]
    
    Redis -->|Cache Hit| JWT[6. Generate JWT]
    Redis -->|Cache Miss| ClarityAPI[Clarity API]
    ClarityAPI -->|User Data| Cache[7. Cache Data]
    Cache --> JWT
    
    JWT -->|8. Return Token| ChatPage
    ChatPage -->|9. User Message| Middleware
    
    Middleware -->|10. Verify JWT| Valid{Valid?}
    Valid -->|No| ErrorAuth[❌ 401 Error]
    Valid -->|Yes| ChatService[Chat Service]
    
    ChatService -->|11. Call AI| ClaudeAPI[Claude API]
    ClaudeAPI -->|12. Need Data| MCPServer[MCP Server]
    
    MCPServer -->|13. Check Permission| PermService[Permission Service]
    PermService -->|14. Validate| Redis
    PermService -->|15. Allowed| ClarityAPI
    
    ClarityAPI -->|16. Data| MCPServer
    MCPServer -->|17. Format| ClaudeAPI
    ClaudeAPI -->|18. Generate Response| ChatService
    
    ChatService -->|19. Stream| ChatPage
    ChatService -->|20. Save| PostgreSQL[(PostgreSQL)]
    
    ChatPage -->|21. Display| User
    
    style User fill:#e1f5ff
    style Middleware fill:#fff4e6
    style ChatService fill:#f3e5f5
    style Redis fill:#ffccbc
    style PostgreSQL fill:#d1c4e9
    style ClaudeAPI fill:#c8e6c9
    style MCPServer fill:#a5d6a7
    style ErrorAuth fill:#ffcdd2
```

# ============================================
# END OF DIAGRAMS
# ============================================

# How to use these diagrams:
# 
# 1. GitHub/GitLab:
#    - Just paste the code blocks in your README.md
#    - They will render automatically
#
# 2. Draw.io:
#    - Go to draw.io
#    - File > Import > Select "Mermaid"
#    - Paste the diagram code
#
# 3. VSCode:
#    - Install "Mermaid Preview" extension
#    - Open this file and use preview
#
# 4. Online Mermaid Editors:
#    - https://mermaid.live
#    - Paste code and export as PNG/SVG
#
# 5. Notion:
#    - Embed using /code block with language "mermaid"
#
# 6. Obsidian:
#    - Supports Mermaid natively in code blocks
