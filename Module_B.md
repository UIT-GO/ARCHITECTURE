```mermaid
flowchart TB
    %% --- Styles ---
    classDef external fill:#f5f5f5,stroke:#333,stroke-width:1px;
    classDef component fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;
    classDef db fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef danger stroke:#d32f2f,stroke-width:2px,stroke-dasharray: 5 5;

    %% --- Actors ---
    User[Customer / Driver App]:::external

    %% --- Legacy Infrastructure (Single Zone) ---
    subgraph Single_Zone ["❌ Single Availability Zone (No Redundancy)"]
        direction TB
        
        %% Direct entry (No ALB)
        APIGW(API Gateway):::component
        
        %% Single Instances
        UserSvc[UserService]:::component
        TripSvc[TripService]:::component
        DriverSvc[DriverService]:::component

        %% Single Databases
        PG[("PostgreSQL")]:::db
        Mongo[("MongoDB")]:::db
        Redis[("Redis")]:::db
    end

    %% --- Flows ---
    User --> APIGW
    
    %% Direct Routing
    APIGW --> UserSvc
    APIGW --> TripSvc
    APIGW --> DriverSvc

    %% DB Connections
    UserSvc --> PG
    TripSvc --> Mongo
    DriverSvc --> Redis

    %% --- Annotations for Weaknesses ---
    User -. "Direct Access\n(No Load Balancer)" .-> APIGW
    TripSvc -. "SPOF\n(Single Point of Failure)" .- Mongo

    class Single_Zone danger;
