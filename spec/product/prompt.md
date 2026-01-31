You are acting as a Senior Product Architect and Tech Lead.

Your task is to convert the following raw product context into a
clear, structured Product Understanding Document that can be shared
with Engineering, Design, QA, and Stakeholders.

### Output Expectations
Create a well-written document with the following sections:
1. Product Overview
2. Business Objective & Success Metrics
3. Target Users & User Personas
4. Core User Journeys (step-by-step)
5. Functional Requirements (Must / Should / Nice-to-have)
6. Non-Functional Requirements
7. Key Assumptions & Constraints
8. Out-of-Scope Items
9. Open Questions / Risks
10. Dependencies (internal & external)

Use concise language, bullet points where applicable, and remove ambiguity.
If any information is missing, infer reasonably and list it under assumptions.

---

### Raw Product Context (from me)

Product Name:
Travel Management System

Problem Statement:
* Used to manage the leads from different sources and convert them into customers.
* Booking from the leads using itenary and packages
* Manage the bookings and payments
* Manage the agents and their commissions
* Should need a dashboard to track total itenary, total leads, converted leads & payments status.


Business Context:
* This product is used to manage the leads from different sources and convert them into customers.
* It is used to manage the bookings and payments
* It is used to manage the agents and their commissions
* It is used to track the leads and bookings

Target Users:
* Admin
* Agents

User Scenarios:
* Admin should be able to create a new itenary and package
* Admin should be able to add agents and their commissions
* Admin should be able to track the leads and bookings
* Admin should be able to manage the payments
* Admin should be able to create Agents
* Admin should be able to do all things that Agents do

* Agent should be able to create leads
* Agent should be able to create bookings for the leads
* Agent should be able to track the leads and bookings
* Agent should be able to manage the payments   
* Agent should be able to support by creating the support tickets & manage the tickets

Key Features:
* Dashboard
* Leads Management
* Bookings Management
* Packages Management
* Quotations Management
* Itinerary Management
* Payments Management
* Agents Management
* Support Tickets Management

Constraints:
* Backend: Express JS
* Frontend: Next JS
* Database: MySQL
* AWS: S3, EC2, Amplify

Timeline / Phase:
* MVP

Additional Notes:
* There are gaps in the requirements, please fill them with reasonable assumptions.

* and also tell me which some tech stacks missed inbetween