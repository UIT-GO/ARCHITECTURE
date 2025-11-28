```mermaid
flowchart LR
    A[Customer] --> B(ALB)
    B --> C(API Gateway)
    C --> D[TripService - AZ A]
    C --> E[TripService - AZ B]
```
