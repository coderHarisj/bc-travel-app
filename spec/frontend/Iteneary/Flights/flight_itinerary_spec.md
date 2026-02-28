# Flight Itinerary – Frontend Technical Specification (Desktop)

## 0. Component Reuse & Impact Analysis

**Reused Components**
- `PageHeader` (Title and back navigation)
- `Card` (Container for flight segments, passenger details, and fare breakdown)
- `Button` (Primary for Download/Share/Cancel actions)
- `Typography` (Standard typography system)
- `Divider` (Vertical and horizontal separators)
- `Icon` (Flight path, baggage, user, seat)

**Extended Components**
- `Timeline` (Modified to support flight segment dots and vertical connectors)

**New Components**
- `FlightSegmentCard` (Layout for Origin, Destination, Airline, Duration, and Stops)
- `FareBreakdownTable` (Read-only list of fare items and total sum)

**Layout Impact**
- Adheres to standard container layouts per [.rules/ui-system.md](file:///e:/POC/bc-travel-app/.rules/ui-system.md).
- Desktop typical split: Main content column (70%) and Side summary column (30%) if applicable.

**State Architecture Impact**
- Read-only representation of booking data; local state for modals (Cancel, Share).

**Design Token Impact**
- No new tokens required; mapped to existing colors and spacing tokens.

## 1. Screen Overview

**Purpose**
- Display the complete finalized details of a flight booking, including itinerary, passenger details, baggage rules, and cost breakdown.

**User Role Context**
- The traveler or the travel administrator viewing a confirmed booking.

**Navigation Entry Points**
- 'My Trips' or 'Bookings' list view.
- Success confirmation page post-booking.

**Exit Paths**
- 'Back' button to booking list.
- Modals for actions (Cancel Flight).

**Dependencies**
- Flight Booking Itinerary API payload.

## 2. Desktop Layout Structure

**Grid Structure**
- Uses desktop 12-column grid structure (e.g., 8 cols for main flow, 4 col for fare breakdown/actions on the right).

**Section Breakdown**
1. **Header:** Title (e.g., "Flight Itinerary"), Booking Reference (PNR), and Status Badge (e.g., "Confirmed").
2. **Flight Segments:** Chronological visualization of the flight path.
3. **Passenger Info:** Table or list with name, age, gender, seat, and meal preferences.
4. **Baggage Rules:** Cabin and check-in baggage allowances per segment.
5. **Fare Summary & Actions:** Base fare, taxes/fees, total amount, plus primary actions (Download, Share, Cancel).

**Component Hierarchy**
- `PageContainer`
  - `Header`
  - `GridContainer`
    - `MainColumn`
       - `FlightSegmentCard` (multiple)
       - `Card` (Passengers)
       - `Card` (Baggage)
    - `SideColumn`
       - `Card` (Fare Breakdown)
       - `ButtonList` (Actions)

**Scroll Behavior**
- Standard vertical scroll.

**Modal vs Full Page**
- Full page view. Actions (like Cancellation confirmation) will open in Center Modals.

## 3. Design System Extraction

**Color Tokens**
- Background: `bg-surface-default` (Page level), `bg-surface-card` (Card level)
- Text: `text-primary` (Titles), `text-secondary` (Subtitles, metadata)
- Status: `color-success-main` (Confirmed) or `color-error-main` (Cancelled)
- Dividers: `border-divider`

**Typography Scale**
- **H1 / H2:** Screen Title
- **H4 / Subtitle1:** Section Titles ("Flight Details", "Passengers")
- **Body1:** Important info (City names, Passenger names, Dates)
- **Body2 / Caption:** Auxiliary info (Airport names, Flight numbers, duration)

**Spacing System**
- Section gap: `gap-xl` (32px) or `gap-lg` (24px).
- Internal card padding: `p-md` (16px) or `p-lg` (24px).

**Border Radius**
- `rounded-lg` or `rounded-xl` for cards and buttons.

**Elevation / Shadow**
- Subtle `shadow-sm` for cards; `shadow-lg` for modals.

## 4. Field-Level Specification

| Field / UI Element | Type | Status | Details |
| --- | --- | --- | --- |
| PNR / Booking ID | Read-only Text | Mandatory | Displayed prominently near header. |
| Status Badge | Badge | Mandatory | Colored based on state (Confirmed/Cancelled). |
| Airline Logo | Image | Mandatory | Airline branding icon. |
| Airline Name & Flight No. | Read-only Text | Mandatory | e.g., "Indigo 6E-123". |
| Departure/Arrival Time | Read-only Text | Mandatory | Formatted Time (e.g., "10:30 AM"). |
| Origin/Destination City | Read-only Text | Mandatory | e.g., "New Delhi (DEL)". |
| Airport/Terminal Info | Read-only Text | Mandatory | e.g., "Terminal 3, Indira Gandhi Int'l". |
| Duration / Layovers | Read-only Text | Mandatory | e.g., "2h 15m (Non-stop)". |
| Passenger Name | Read-only Text | Mandatory | Full name of the traveler. |
| Seat / Meal Preference | Read-only Text | Optional | Appears if selected during booking. |
| Baggage Allowance | Read-only Text | Mandatory | E.g., "15 Kg Check-in, 7 Kg Cabin". |
| Fare Items (Base, Taxes) | Read-only Text | Mandatory | Currency formatted. |
| Total Amount | Read-only Text (Bold)| Mandatory | Currency formatted grand total. |

## 5. Interaction & UX Behavior

**Loading States**
- **Initial Load:** Skeleton screens approximating the card layouts (Header skeleton, Flight route skeleton).
- **Button Actions:** Inline circular progress indicators on primary action buttons (e.g., while downloading PDF).

**Standard States**
- No form validations needed as this is a read-only view.
- Error banners: If API fetch fails or "Download" fails, show toast/banner notification.

**Empty States**
- If an optional data point (e.g., Meal Preference) is empty, do not show the label, or display "Not Specified".
- If the itinerary has no layovers, don't show the layover divider state.

---

## 6. Accessibility Specification

- **WCAG Considerations:** Ensure accurate `alt` text for airline logos and icons.
- **Keyboard Navigation Flow:** Focus should move intuitively from top header to actions, and down through interactive elements.
- **Screen Reader Requirements:** Use semantic data tables or visually hidden text for fare breakdowns and passenger details.
- **ARIA Requirements:** Use `aria-label` for icon-only buttons (like Share or Download). Alert roles for error banners.
- **Focus Management:** Modals (like Cancel Flight) must trap focus until dismissed or confirmed.
- **Contrast Compliance:** All text and critical icons must meet WCAG AA standards (4.5:1 ratio).

---

## 7. State Management Plan

- **Local State:** Modal visibility (open/close state), local loading states for actions like "Downloading PDF".
- **Global State (Redux or equivalent):** Authentication state. Itinerary data may be cached globally to prevent refetching during same-session navigation.
- **Derived State:** Date/Time formatting based on raw timestamps. Total travel time calculation (if not provided).
- **Async Handling:** Fetching itinerary detail via API call on component mount (`useEffect` or specific data-fetching hook).
- **Error State Handling:** Catch block updating local error state to show banners/toasts. Re-try mechanism if network drops.
- **Form Isolation Strategy:** Any active input (like Cancellation reason or Email Share fields) should be isolated within their respective modal components.

---

## 8. API Integration Requirements (Only If Inferable)

- **Expected Operation Type:** `GET` request for reading itinerary data; `POST` or `DELETE` for generating cancellation requests.
- **Trigger Points:** On page load (`GET` id/PNR from route parameters). On Modal Confirm (`POST`).
- **Payload Structure (only if clearly implied):** For 'Share', implies `{"email": "string"}`. For 'Cancel', implies `{"bookingId": "string", "reason": "string"}`.
- **Backend Clarifications Required:** Will the backend supply durations natively, or must the frontend calculate the difference between departure and arrival timestamps?

---

## 9. Performance Considerations (Desktop POC)

- **Rendering Complexity:** Low rendering complexity.
- **Re-render Risks:** Opening local modals shouldn't re-render the entire itinerary heavy-components.
- **Virtualization Needs:** Not needed (passenger and segment arrays are typically small).
- **Debounce Requirements:** Not needed (no search features).
- **Memoization Considerations:** `FlightSegmentCard` and `FareBreakdownTable` can be wrapped in `React.memo` since they are read-only and static once loaded.

---

