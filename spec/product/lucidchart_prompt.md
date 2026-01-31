# LucidChart AI Prompt: Travel Management System Flow

**Instruction:** 
Copy and paste the following prompt into LucidChart's "Generate Diagram" feature (or use it as a guide for manual creation) to visualize the system architecture and user flows.

---

## Prompt for LucidChart

**Subject:** Travel Management System - User Flow & Entity Relationship Diagram

**Description:**
Create a comprehensive flow diagram for a B2B2C Travel Management System. The system involves two key actors: **Admin** and **Agent**.

**Please include the following components and flows:**

1.  **Actors:**
    *   **Admin:** The system owner/manager.
    *   **Agent:** External travel agents selling packages.

2.  **Key Entities:**
    *   **Dashboard** (Admin & Agent views)
    *   **Lead** (Source, Status, Contact Info)
    *   **Itinerary/Package** (Destinations, Details, Price)
    *   **Booking** (Linked to Lead + Agent + Package)
    *   **Payment** (Status, Amount, Ref ID)
    *   **Commission** (Calculated for Agent)
    *   **Support Ticket** (Issues raised by Agent)

3.  **Process Flows:**
    *   **Admin Setup Flow:** Admin -> Creates Agency/Agent -> Sets Commission Rate -> Creates Standard Packages.
    *   **Sales Flow (Agent):** Agent -> Adds Lead -> Creates Custom Itinerary (or selects Package) -> Sends Quote -> Lead Accepts -> Agent converts to Booking.
    *   **Payment & Commission Flow:** Booking Confirmed -> Payment Recorded (Admin/Agent) -> System updates Payment Status -> System calculates Commission -> Updates Agent's Earnings Wallet.
    *   **Support Flow:** Agent -> Raises Ticket -> Admin -> Reviews & Resolves.

4.  **Relationships:**
    *   One **Agent** manages many **Leads**.
    *   One **Lead** can have multiple **Itineraries** (Quotes).
    *   One **Booking** is linked to one **Lead** and one **Package/Itinerary**.
    *   One **Booking** has many **Payments**.

**Visual Style:** 
*   Use Swimlanes for "Admin", "Agent", and "System/Backend".
*   Use Diamond shapes for decision points (e.g., "Is Payment Complete?", "Is Lead Interested?").
*   Use Rectangles for process steps/actions.
*   Connect with directional arrows showing the data flow.

---
