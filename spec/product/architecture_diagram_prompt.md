# LucidChart AI Prompt: System Architecture Diagram

**Instruction:**
Copy and paste the following prompt into LucidChart's "Generate Diagram" feature to visualize the technical architecture of the system.

---

## Prompt for LucidChart

**Subject:** Travel Management System - High-Level Technical Architecture

**Description:**
Create a detailed System Architecture Diagram for a web-based Travel Management System using the MERN stack (MySQL variant) on AWS.

**Layers to Include:**

1.  **Client Layer (Frontend):**
    *   **Dashboard Component:** Built with **Next.js**.
    *   **Deployment:** Hosted on **AWS Amplify**.
    *   **Actors:** Admin and Agent browsers connecting via HTTPS.

2.  **Network / Security Layer:**
    *   **Load Balancer / Gateway:** Nginx or AWS Application Load Balancer (ALB).
    *   **Firewall:** Handling secure traffic to the backend.

3.  **Application Layer (Backend):**
    *   **Server:** **EC2 Instance** running **Node.js / Express.js**.
    *   **Services:**
        *   Lead Service
        *   Booking Service
        *   Payment Service
        *   Commission Engine
        *   Notification Service (SES worker)

4.  **Data Layer:**
    *   **Database:** **MySQL** (RDS Instance).
    *   **Storage:** **AWS S3** (for storing PDF itineraries, images, and documents).

5.  **External Integrations:**
    *   **Payment Gateway:** (Abstract external system).
    *   **Email Service:** **AWS SES**.

**Connections:**
*   Users access **AWS Amplify** (Next.js) -> Calls **API (EC2/Express)** over HTTPS.
*   **Express Server** reads/writes to **MySQL RDS**.
*   **Express Server** stores/retrieves files from **S3**.
*   **Express Server** triggers emails via **SES**.

**Visual Style:**
*   Use standard AWS icons for AWS services (EC2, S3, Amplify, RDS, SES).
*   Group components into "Frontend", "Backend VPC", and "Data Persistence" boxes.
*   Use solid lines for synchronous API calls and dashed lines for asynchronous background tasks.

---
