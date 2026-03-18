# LinkVault Architecture Diagrams

This document contains all Mermaid diagrams for the LinkVault URL Shortener project. These diagrams are designed to be rendered on GitHub (native Mermaid support) and in documentation platforms.

%%{init: {'theme': 'neutral'}}%%

## System Architecture

```mermaid
graph TB
    subgraph Client["Client Tier"]
        Browser["Browser<br/>(User)"]
        Admin["Admin Portal<br/>(Blazor Server)"]
    end

    subgraph AzureEdge["Azure Edge Layer"]
        FrontDoor["Azure Front Door<br/>(Global Load Balancer)"]
        WAF["WAF<br/>(OWASP Protection)"]
    end

    subgraph AzureCompute["Azure Compute"]
        BlazorApp["LinkVault.BlazorApp<br/>(App Service - Blazor Server)"]
        ApiApp["LinkVault.Api<br/>(App Service - Minimal APIs)"]
        FuncApp["LinkVault.Functions<br/>(Azure Functions)"]
    end

    subgraph AzureData["Azure Data Layer"]
        SqlDb["Azure SQL Database<br/>(Primary)"]
        SqlDbReplica["Azure SQL Database<br/>(Geo-Replica)"]
        Redis["Azure Cache for Redis<br/>(Cache Layer)"]
    end

    subgraph External["External Services"]
        Github["GitHub OAuth<br/>(Authentication)"]
        QrApi["QR Code API<br/>(QRServer.com)"]
        PreviewApi["Link Preview API<br/>(Microlink.io)"]
    end

    Browser -->|"HTTPS<br/>/s/{code}"| FrontDoor
    Admin -->|"HTTPS"| BlazorApp
    FrontDoor -->|"Route<br/>/api/*"| ApiApp
    FrontDoor -->|"Route<br/>/*"| BlazorApp
    BlazorApp -->|"API Calls"| ApiApp
    ApiApp -->|"Read/Write"| SqlDb
    ApiApp -->|"Read Replica"| SqlDbReplica
    ApiApp -->|"Cache Lookups"| Redis
    ApiApp -->|"302 Redirect"| Browser
    ApiApp -->|"Track Click"| SqlDb
    FuncApp -->|"Scheduled<br/>Queries"| SqlDb
    FuncApp -->|"Cache<br/>Cleanup"| Redis
    ApiApp -->|"OAuth 2.0"| Github
    ApiApp -->|"GET QR PNG"| QrApi
    ApiApp -->|"Fetch Metadata"| PreviewApi

    style AzureEdge fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style AzureCompute fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style AzureData fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style External fill:#fce4ec,stroke:#c2185b,stroke-width:2px
```

## Container Architecture

```mermaid
graph TB
    subgraph Web["Presentation Layer"]
        BlazorServer["Blazor Server<br/>(Interactive SSR)<br/>linkvault.io"]
        SPA["Blazor WASM<br/>(Progressive Web App)<br/>app.linkvault.io"]
    end

    subgraph API["Application Layer"]
        Gateway["YARP API Gateway<br/>(Routing, Auth)"]
        LinkService["LinkService<br/>(Shorten, Redirect)"]
        AnalyticsService["AnalyticsService<br/>(Click Tracking)"]
        AuthService["AuthService<br/>(JWT, OAuth)"]
        CacheService["CacheService<br/>(Redis)"]
        RateLimitMiddleware["Rate Limiting<br/>(Per-IP, Per-User)"]
    end

    subgraph Data["Data Layer"]
        LinksRepo["ILinkRepository<br/>(EF Core)"]
        ClicksRepo["IClickRepository<br/>(EF Core)"]
        Redis["Redis<br/>(Distributed Cache)"]
        SqlServer["SQL Server<br/>(Primary Storage)"]
        SqlReplica["SQL Server<br/>(Read Replica)"]
    end

    subgraph Functions["Background Processing"]
        ExpirationFunc["LinkExpirationFunction<br/>(Timer: 5 min)"]
        AnalyticsFunc["AnalyticsAggregationFunction<br/>(Timer: 1 hour)"]
    end

    Web --> Gateway
    Gateway --> RateLimitMiddleware
    RateLimitMiddleware --> LinkService
    RateLimitMiddleware --> AnalyticsService
    RateLimitMiddleware --> AuthService
    LinkService --> CacheService
    AnalyticsService --> CacheService
    CacheService --> Redis
    LinkService --> LinksRepo
    AnalyticsService --> ClicksRepo
    LinksRepo --> SqlServer
    ClicksRepo --> SqlServer
    AnalyticsService --> SqlReplica
    FuncApp --> LinksRepo
    FuncApp --> CacheService

    style Web fill:#e3f2fd
    style API fill:#e8f5e9
    style Data fill:#fff3e0
    style Functions fill:#f3e5f5
```

## Redirect Flow Sequence

```mermaid
sequenceDiagram
    participant User as User Browser
    participant FD as Azure Front Door
    participant Api as LinkVault.Api
    participant RL as Rate Limiter
    participant Cache as Redis
    participant DB as SQL Server
    participant Track as Click Tracker

    User->>FD: GET /s/abc123
    FD->>Api: Route to nearest region
    Api->>RL: Check rate limits
    RL-->>Api: Allowed

    Api->>Cache: GET link:abc123
    Cache-->>Api: null (cache miss)

    Api->>DB: SELECT * FROM Links<br/>WHERE Code = 'abc123'
    DB-->>Api: Link entity

    alt Link found and valid
        Api->>Cache: SETEX link:abc123 300 [data]
        Api->>Track: Record click event
        Api-->>User: 302 Redirect to originalUrl
    else Link expired
        Api-->>User: 410 Gone
    else Link not found
        Api-->>User: 404 Not Found
    end

    User->>FD: GET /s/abc123 (within 5 min)
    FD->>Api: Route to nearest region
    Api->>Cache: GET link:abc123
    Cache-->>Api: Link entity (cache hit)
    Api->>Track: Record click event
    Api-->>User: 302 Redirect
```

## URL Shortening Flow

```mermaid
sequenceDiagram
    participant User as User (Blazor UI)
    participant Blazor as Blazor Server
    participant Api as LinkVault.Api
    participant Auth as AuthService
    participant LinkService as LinkService
    participant IDGen as IdGenerator
    participant Cache as Redis
    participant DB as SQL Server

    User->>Blazor: Submit URL + Custom Alias
    Blazor->>Api: POST /api/links { url, customAlias }
    Api->>Auth: Validate JWT + User
    Auth-->>Api: User authenticated

    alt Custom alias provided
        Api->>DB: Check Code exists
        DB-->>Api: Available
    else No custom alias
        Api->>IDGen: Generate ULID
        IDGen-->>Api: 01ARZ3NDEKTSV4RRFFQ69G5FAV
        Api->>IDGen: Encode to Base62
        IDGen-->>Api: abc123XYZ
    end

    Api->>LinkService: CreateShortLinkAsync(url, code)
    LinkService->>DB: INSERT INTO Links
    DB-->>LinkService: Link created

    LinkService->>Cache: SET link:abc123 [data] EX 300
    Cache-->>LinkService: Cached

    LinkService-->>Api: ShortLinkResponse { shortUrl, qrCodeUrl }
    Api-->>Blazor: 201 Created
    Blazor-->>User: Display short URL + QR code
```

## Analytics Dashboard Flow

```mermaid
sequenceDiagram
    participant User as User Dashboard
    participant Blazor as Blazor Server
    participant Api as LinkVault.Api
    participant Analytics as AnalyticsService
    participant Cache as Redis
    participant DB as SQL Server

    User->>Blazor: Navigate to Dashboard
    Blazor->>Api: GET /api/links/my-links
    Api->>Analytics: GetUserLinksAsync(userId)
    Analytics->>Cache: GET user:{id}:links
    Cache-->>Analytics: null (cache miss)

    Analytics->>DB: SELECT Links.*,<br/>COUNT(ClickEvents.Id) AS TotalClicks<br/>FROM Links<br/>LEFT JOIN ClickEvents ON...
    DB-->>Analytics: Aggregated link data

    Analytics->>Cache: SETEX user:{id}:links 120 [json]
    Analytics-->>Api: LinkStatsResponse[]
    Api-->>Blazor: JSON response
    Blazor-->>User: Render Dashboard

    User->>Blazor: View link analytics
    Blazor->>Api: GET /api/links/{id}/analytics?period=7d
    Api->>DB: SELECT * FROM ClickEvents<br/>WHERE LinkId = @id<br/>AND ClickedAt > @startDate
    DB-->>Api: Click events
    Api-->>Blazor: Analytics data
    Blazor-->>User: Render Charts
```

## Database Schema (ER Diagram)

```mermaid
erDiagram
    AspNetUsers ||--o{ Links : "creates"
    AspNetUsers {
        string Id PK "AspNetUsers"
        string Email UK "Unique email"
        string UserName UK "Unique username"
        string NormalizedEmail "Normalized"
        string NormalizedUserName "Normalized"
        string PasswordHash "Hashed password"
        datetimeoffset CreatedAt "Creation date"
        datetimeoffset LastLoginAt "Last login"
        bool IsActive "Account status"
    }

    Links ||--o{ ClickEvents : "tracks"
    Links ||--o| AspNetUsers : "belongs to"
    Links {
        guid Id PK "Primary Key"
        string Code UK "Unique short code (indexed)"
        string OriginalUrl "Long URL (required)"
        string Title "Optional title"
        string Description "Optional description"
        bool IsCustom "Custom alias flag"
        datetimeoffset ExpiresAt "Expiration (nullable)"
        int MaxClicks "Click limit (nullable)"
        int ClickCount "Current click count"
        int Status "0=Active, 1=Expired, 2=Disabled"
        datetimeoffset CreatedAt "Creation timestamp"
        string CreatedByUserId FK "Nullable, links to AspNetUsers"
        bool IsPublic "Public dashboard visibility"
        datetimeoffset? UpdatedAt "Last update time"
    }

    ClickEvents {
        guid Id PK "Primary Key"
        guid LinkId FK "Links.Id (indexed)"
        datetimeoffset ClickedAt "Click timestamp (indexed)"
        string IpAddress "IPv4/IPv6"
        string UserAgent "Browser string"
        string ReferrerUrl "HTTP referrer (nullable)"
        string Country "GeoIP country"
        string Region "GeoIP region/state"
        string City "GeoIP city"
        string DeviceType "Desktop/Mobile/Tablet/Bot"
        string BrowserName "Browser name"
        string BrowserVersion "Browser version"
        string OsName "OS name"
        string OsVersion "OS version"
        bool IsUnique "First click from this IP"
    }

    AspNetUsers ||--o{ ExternalLogins : "has linked"
    ExternalLogins {
        string LoginProvider PK "GitHub/Google/Microsoft"
        string ProviderKey PK "Provider user ID"
        string? DisplayName "Display name"
        string UserId FK "AspNetUsers.Id"
    }

    Links {
        INDEX idx_links_code ON Links(Code)
        INDEX idx_links_user ON Links(CreatedByUserId)
        INDEX idx_links_status ON Links(Status)
        INDEX idx_links_expires ON Links(ExpiresAt) WHERE Status = 0
    }

    ClickEvents {
        INDEX idx_clicks_link ON ClickEvents(LinkId)
        INDEX idx_clicks_time ON ClickEvents(ClickedAt)
        INDEX idx_clicks_geo ON ClickEvents(Country, City)
        INDEX idx_clicks_device ON ClickEvents(DeviceType)
    }
```

## Rate Limiting Architecture

```mermaid
flowchart TB
    subgraph Incoming["Incoming Request"]
        Req["HTTP Request<br/>(IP, JWT, Endpoint)"]
    end

    subgraph Limiters["Rate Limiters (Middleware)"]
        GlobalLimiter["Global Limiter<br/>(5000 req/min)"]
        IpLimiter["IP Limiter<br/>(100 req/min per IP)"]
        UserLimiter["User Limiter<br/>(1000 req/min per user)"]
        EndpointLimiter["Endpoint Limiter<br/>(Per-endpoint config)"]
    end

    subgraph Decisions["Rate Limit Decisions"]
        Check1["Check Global"]
        Check2["Check IP"]
        Check3["Check User"]
        Check4["Check Endpoint"]
    end

    subgraph Actions["Actions"]
        Allow["Allow Request<br/>(Add rate limit headers)"]
        Reject["429 Too Many Requests<br/>(Retry-After header)"]
        Log["Log Rate Limit Event<br/>(Application Insights)"]
    end

    Req --> GlobalLimiter
    GlobalLimiter --> Check1
    Check1 -->|"Pass"| IpLimiter
    Check1 -->|"Exceeded"| Reject
    IpLimiter --> Check2
    Check2 -->|"Pass"| UserLimiter
    Check2 -->|"Exceeded"| Reject
    UserLimiter --> Check3
    Check3 -->|"Pass"| EndpointLimiter
    Check3 -->|"Exceeded"| Reject
    EndpointLimiter --> Check4
    Check4 -->|"Pass"| Allow
    Check4 -->|"Exceeded"| Reject

    Reject --> Log
    Allow --> Log

    style GlobalLimiter fill:#e3f2fd
    style IpLimiter fill:#e8f5e9
    style UserLimiter fill:#fff3e0
    style EndpointLimiter fill:#fce4ec
    style Reject fill:#ffcdd2,stroke:#c62828
    style Allow fill:#c8e6c9,stroke:#2e7d32
```

## Azure Functions

### Link Expiration Function

```mermaid
stateDiagram-v2
    [*] --> Scheduled

    Scheduled --> QueryExpired : Timer Trigger
    QueryExpired --> FoundExpired : Found expired links?

    FoundExpired --> ProcessExpired : For each expired link
    ProcessExpired --> UpdateStatus : Set Status = Expired
    UpdateStatus --> InvalidateCache : DELETE from Redis
    InvalidateCache --> LogEvent : Log LinkExpired event

    FoundExpired --> LogCompletion : No expired links
    LogEvent --> LogCompletion

    LogCompletion --> Scheduled : Next execution

    note right of QueryExpired
        Query: Status = Active AND
        ExpiresAt < UTCNOW()
        OR (MaxClicks IS NOT NULL AND
        ClickCount >= MaxClicks)
    end note

    note right of UpdateStatus
        Batch update for efficiency
        ExecuteSqlCommand
        Max 1000 per batch
    end note
```

### Click Analytics Aggregation Function

```mermaid
stateDiagram-v2
    [*] --> Scheduled

    Scheduled --> ReadRedisCounters : Timer Trigger (Hourly)
    ReadRedisCounters --> FlushCounters : Read pending click counts

    FlushCounters --> UpdateLinkStats : For each counter
    UpdateLinkStats --> IncrementCounts : UPDATE ClickCount + N

    FlushCounters --> UpdateDailyStats : Aggregate by day/country/device
    UpdateDailyStats --> InsertDailyAgg : INSERT aggregated stats

    ReadRedisCounters --> CompleteCounters : No pending counters
    IncrementCounts --> CompleteCounters
    InsertDailyAgg --> ClearRedisCounters : Delete processed keys

    ClearRedisCounters --> LogAggregation : Log summary
    CompleteCounters --> LogAggregation
    LogAggregation --> Scheduled : Next execution

    note right of ReadRedisCounters
        KEYS analytics:clicks:*
        SCAN for pending counters
    end note

    note right of UpdateDailyStats
        GROUP BY link_id, date,
        country, device_type
        SUM counts
    end note
```

## Technology Stack Summary

```mermaid
mindmap
  root((LinkVault))
    Frontend
      Blazor Server
      Tailwind CSS
      Chart.js
      QR Code Generator
    Backend
      ASP.NET Core 10
      Minimal APIs
      Entity Framework Core
      Redis (StackExchange.Redis)
      Rate Limiting Middleware
    Database
      Azure SQL
      SQL Server 2022
      Geo-Replication
      Read Replicas
    Cache
      Azure Cache for Redis
      Cache-Aside Pattern
      TTL Management
    Security
      ASP.NET Identity
      GitHub OAuth
      JWT Bearer Tokens
      WAF (Azure Front Door)
    Infrastructure
      Azure Front Door
      Azure App Service
      Azure Functions
      Azure SQL
      Azure Cache for Redis
    Observability
      Application Insights
      Serilog
      Azure Monitor
      Log Analytics
    External APIs
      QR Code (QRServer.com)
      Link Preview (Microlink.io)
      GeoIP Service
```
