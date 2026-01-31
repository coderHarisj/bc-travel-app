# LucidChart AI Prompt: AWS Secure VPC Architecture

**Instruction:**
Copy and paste the following prompt into LucidChart's "Generate Diagram" feature to visualize the secure AWS infrastructure.

---

## Prompt for LucidChart

**Subject:** AWS Secure VPC Architecture for Travel Management System

**Description:**
Create a detailed AWS Infrastructure Diagram focusing on a **Secure Single VPC** architecture with **Private Subnets** for the application and database layers.

**Core Requirements:**
1.  **VPC boundary:** Single VPC (e.g., 10.0.0.0/16).
2.  **Public Subnet:** Contains *only* English-facing components (Application Load Balancer, NAT Gateway).
3.  **Private Subnet:** Contains the **Frontend** (Next.js context), **Backend** (Express on EC2), and **Database** (MySQL RDS). All computing resources are hidden here.
4.  **CloudFront:** Sits outside the VPC at the Edge. Connects to the **Application Load Balancer** (Public Subnet) to reach the private backend.

**Detailed Data Flow:**
1.  **User Request** -> **Route 53** -> **CloudFront** (Edge Location with WAF).
2.  **CloudFront** -> **Application Load Balancer** (Public Subnet, acting as the ingress controller).
3.  **ALB** -> **Target Group** -> **EC2 Instances** (Private Subnet).
4.  **EC2 Instances** -> **RDS MySQL** (Private Subnet).
5.  **EC2 Instances** -> **NAT Gateway** (Public Subnet) -> **Internet** (for external API calls/updates).
6.  **S3 Bucket:** Accessed by EC2 via **VPC Endpoint** (Gateway) to keep traffic internal.

**Components to Visualize:**
*   **Region:** AWS Cloud.
*   **VPC:** Box containing subnets.
*   **Public Subnet:** Icons for ALB, NAT Gateway, Bastion Host (Optional).
*   **Private Subnet:** Icons for EC2 (Auto Scaling Group), RDS (MySQL).
*   **Outside VPC:** CloudFront, Route53, S3, Internet Gateway.

**Visual Style:**
*   Use standard AWS isometric icons.
*   Use arrows to show the flow of traffic (Ingress vs Egress).
*   Label the subnets specifically as "Public Subnet (Ingress Only)" and "Private Subnet (App & Data)".

---
