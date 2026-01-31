# AWS Infrastructure Diagram (Mermaid)

This architecture demonstrates a **Secure Cloud Architecture** where backend resources are isolated in private subnets, approached only via an Application Load Balancer and CloudFront.

## Architecture Diagram

```mermaid
graph TB
    %% External Users
    User[End User / Browser]
    
    %% Edge Layer
    subgraph "AWS Edge Location"
        CF[CloudFront Distribution]
        WAF[AWS WAF]
    end

    %% VPC Definition
    subgraph "AWS Region"
        subgraph "VPC (10.0.0.0/16)"
            
            %% Public Subnet
            subgraph "Public Subnet (DMZ)"
                IGW[Internet Gateway]
                ALB[Application Load Balancer]
                NAT[NAT Gateway]
            end

            %% Private Subnet
            subgraph "Private Subnet (App & Data)"
                
                subgraph "Auto Scaling Group"
                    EC2_1[EC2 Instance 1<br>(Node.js/Express)]
                    EC2_2[EC2 Instance 2<br>(Node.js/Express)]
                end
                
                RDS[(MySQL RDS)]
                VPCE_S3[VPC Endpoint<br>(Gateway for S3)]
            end
        end
        
        %% Services Outside VPC but in Region
        S3[S3 Bucket<br>(Static Assets/Builds)]
    end

    %% Access Flow
    User -->|HTTPS| CF
    CF -->|Secure Traffic| WAF
    WAF -->|Forward| ALB
    
    ALB -->|Route traffic| EC2_1
    ALB -->|Route traffic| EC2_2
    
    EC2_1 -->|Read/Write| RDS
    EC2_2 -->|Read/Write| RDS
    
    %% Outbound / Resource Flow
    EC2_1 -->|Private Access| VPCE_S3
    VPCE_S3 -.->|Internal S3 Traffic| S3
    
    EC2_1 -.->|Outbound Updates| NAT
    NAT -.->|Internet Access| IGW
    
    style User fill:#f9f,stroke:#333
    style CF fill:#ff9900,stroke:#333
    style ALB fill:#ff9900,stroke:#333
    style EC2_1 fill:#ff9900,stroke:#333
    style RDS fill:#336699,stroke:#fff,color:#fff
```

## Key Constraint Implementation
*   **Single Private Subnet Logic:** All compute (EC2) and storage (RDS) are inside the `Private Subnet`.
*   **CloudFront Connectivity:** CloudFront connects to the `ALB` in the Public Subnet. The ALB acts as the bridge to the private EC2 instances. This honors the requirement for "CloudFront to connect to... backend servers" while keeping the backend private.
