flowchart TB
    %% --- Styles ---
    classDef region fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef az fill:#fff9c4,stroke:#fbc02d,stroke-dasharray: 5 5;
    classDef db fill:#e8f5e9,stroke:#2e7d32;
    classDef svc fill:#f3e5f5,stroke:#7b1fa2;
    classDef lb fill:#fff3e0,stroke:#e65100;
    classDef external fill:#eeeeee,stroke:#333;

    %% --- External ---
    User[Customer / Driver App]:::external

    %% --- Region Scope ---
    subgraph Region["☁️ Region: Singapore (ap-southeast-1)"]
        direction TB
        
        ALB(Application Load Balancer):::lb
        APIGW(API Gateway):::lb
        
        %% --- Availability Zone A ---
        subgraph AZ_A["Availability Zone A"]
            direction TB
            UserSvcA[UserService - A]:::svc
            TripSvcA[TripService - A]:::svc
            DriverSvcA[DriverService - A]:::svc
            
            subgraph Data_A["Data Layer (Primary)"]
                PG_Pri[("Postgres (Primary)")]:::db
                Mongo_Pri[("MongoDB (Primary)")]:::db
                Redis_A[("Redis Node A")]:::db
            end
        end

        %% --- Availability Zone B ---
        subgraph AZ_B["Availability Zone B"]
            direction TB
            UserSvcB[UserService - B]:::svc
            TripSvcB[TripService - B]:::svc
            DriverSvcB[DriverService - B]:::svc

            subgraph Data_B["Data Layer (Standby)"]
                PG_Sec[("Postgres (Standby)")]:::db
                Mongo_Sec[("MongoDB (Secondary)")]:::db
                Redis_B[("Redis Node B")]:::db
            end
        end
    end

    %% --- Connectivity Flows ---
    User --> ALB
    ALB --> APIGW
    
    %% Traffic Distribution
    APIGW --> UserSvcA & UserSvcB
    APIGW --> TripSvcA & TripSvcB
    APIGW --> DriverSvcA & DriverSvcB

    %% Service Communication & Circuit Breaker
    TripSvcA -- "⚡ Circuit Breaker" --> DriverSvcA
    TripSvcB -- "⚡ Circuit Breaker" --> DriverSvcB

    %% Database Connections (Compute to Data)
    UserSvcA & UserSvcB --> PG_Pri
    TripSvcA & TripSvcB --> Mongo_Pri
    DriverSvcA & DriverSvcB --> Redis_A & Redis_B

    %% Data Replication Flows
    PG_Pri <== "Synchronous Rep" ==> PG_Sec
    Mongo_Pri <== "Replica Set" ==> Mongo_Sec
    Redis_A <== "Cluster Mesh" ==> Redis_B

    class Region region;
    class AZ_A,AZ_B az;
