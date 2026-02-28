## 1. Executive Summary

### Purpose
To display the complete and finalized details of a flight booking in a read-only, highly readable desktop view. The page will present the itinerary, passenger details, baggage rules, and complete cost breakdown clearly and intuitively without relying on hardcoded configuration.

### Context & Flow
- **User Role Context:** The traveler or the travel administrator viewing a confirmed/cancelled booking.
- **Entry Points:** 'My Trips', 'Bookings' list view, or post-booking confirmation redirect.
- **Exit Paths:** 'Back' navigation to list, or Modals for actions like Cancellation.

---

## 2. Architecture & Folder Structure

Following the `.rules/architecture.md` and feature-based modular principles, the flight itinerary components will reside entirely within the itinerary feature folder. 

```text
src/
└── features/
    └── itinerary/
        ├── pages/
        │   └── FlightItineraryPage.jsx       # Main orchestration page
        ├── components/
        │   ├── FlightSegmentCard.jsx         # Origin/Dest/Time/Airline
        │   ├── PassengerDetailsCard.jsx      # Read-only table of passengers
        │   ├── BaggageRulesCard.jsx          # Cabin/Check-in allowance
        │   └── FareBreakdownCard.jsx         # Base fare, taxes, total
        ├── services/
        │   └── itineraryService.js           # Protected Axios calls
        └── utils/
            └── itineraryFormatters.js        # Date/Time/Currency formatters
```

---

## 3. Desktop Layout Structure

Following `.rules/ui-system.md`, the layout strictly utilizes the MUI Grid system and avoiding fixed widths.

### Grid Structure (`lg` breakpoint)
- Uses a standard desktop 12-column grid.
- **Main Content Column (8 columns):** Contains chronological Flight Segments, Passenger Details, and Baggage Rules.
- **Side Summary Column (4 columns):** Contains Fare Breakdown and Primary Action Buttons (sticky or static).

### Component Hierarchy
```html
<PageContainer>
  <PageHeader title="Flight Itinerary" pnr="{PNR}" status="{Status}" />
  <GridContainer container spacing={3}>
    
    <!-- Left: 8 Columns -->
    <GridItem item xs={12} lg={8}>
       <FlightSegmentCard />  <!-- Repeated per segment -->
       <PassengerDetailsCard />
       <BaggageRulesCard />
    </GridItem>
    
    <!-- Right: 4 Columns -->
    <GridItem item xs={12} lg={4}>
       <FareBreakdownCard />
       <ActionButtonsList />
    </GridItem>
    
  </GridContainer>
</PageContainer>
```

---

## 4. Component Decomposition 

Adhering to `.rules/component-guidelines.md` and `.rules/coding-standards.md`, components are segregated into Shared and Feature-Specific.

### Reused Shared Components (`src/shared/components/`)
- `Card`: Surface container for all detailed sections.
- `PageHeader`: Consistent title and back navigation layout.
- `Typography`: Standardized text elements following the theme.
- `Divider`: Vertical and horizontal layout separators.
- `Icon`: SVGs for flight path, baggage, user, and seats.
- `Button`: Primary/Secondary actions without inline logic.

### Extended Components
- `Timeline`: Modified to support flight segment dots and vertical layover connectors.

### New Feature Components (`src/features/itinerary/components/`)
- `FlightSegmentCard`: Renders Airline Logo, Flight Number, Origin/Destination Locations, Terminal Info, Departure/Arrival Times, and Duration.
- `PassengerDetailsCard`: Renders read-only list of passengers with their pre-selected seats and meals.
- `BaggageRulesCard`: Renders checked and cabin baggage capacity.
- `FareBreakdownCard`: Displays Base Fare, Taxes, Add-ons, and Grand Total.

---

## 5. Design System & Theme Integration

Extracted following `.rules/ui-system.md` constraints (No magic numbers, no hardcoded colors):

### Color Tokens (MUI Palette mapping)
- **Backgrounds:** `theme.palette.background.default` (Page), `theme.palette.background.paper` (Cards).
- **Text:** `theme.palette.text.primary` (Headings), `theme.palette.text.secondary` (Timestamps, metadata).
- **Status Badges:** `theme.palette.success.main` (Confirmed), `theme.palette.error.main` (Cancelled/Failed).
- **Borders:** `theme.palette.divider`.

### Typography Scale (MUI Typography mapping)
- `h4` / `h5`: Page Titles and Status.
- `h6` / `subtitle1`: Section Titles ("Flight Details", "Fare Breakdown").
- `body1`: Primary information (City names, Passenger names).
- `body2`: Secondary information (Duration, Airport names, specific dates).
- `caption`: Auxiliary info (Taxes description, Seat assignment labels).

### Spacing & Borders
- **Grid Gap:** Uses `theme.spacing(3)` (24px) for desktop margins/gaps.
- **Padding:** Card interiors use `theme.spacing(2)` or `theme.spacing(3)`.
- **Border Radius:** Mapped to standard `shape.borderRadius` (e.g., `rounded-lg`).
- **Elevation:** Flat design with subtle `elevation={1}` for cards.

---

## 6. Field-Level Specification & Data Mapping

All fields are read-only. Display is contingent on backend availability.

| UI Element | Type | Status | Rendering Requirement |
| :--- | :--- | :--- | :--- |
| **PNR / Booking ID** | Read-only Text | Mandatory | Highlighted prominently in PageHeader |
| **Status Badge** | Chip/Badge | Mandatory | Color mapped to booking state enum |
| **Airline Logo** | Image | Mandatory | Standardized dimensions, alt-text applied |
| **Airline & Flight No.**| Read-only Text | Mandatory | e.g. "Indigo 6E-123" |
| **Locations** | Read-only Text | Mandatory | Origin and destination city + airport code |
| **Timestamps** | Read-only Text | Mandatory | Departure/Arrival time natively formatted |
| **Duration / Layovers** | Read-only Text | Mandatory | e.g. "2h 15m (1 Stop)" |
| **Terminal Info** | Read-only Text | Optional | Render if terminal data is present |
| **Passenger Name** | Read-only Text | Mandatory | Full String |
| **Seat / Meal** | Read-only Text | Optional | Omitted visually if unselected |
| **Baggage Allowance** | Read-only Text | Mandatory | Format: "X Kg Check-in \| Y Kg Cabin" |
| **Fare breakdown** | Currency Text | Mandatory | Formatted via shared utility |

---

## 7. State Management Implementation

Strict compliance with `.rules/state-management.md`.

### Redux State
- **NOT USED** for local page interactions.
- Accessing shared authentication token for the localized API fetch only.

### Local State (React `useState`)
- **Modals:** Boolean flags for toggling `CancelConfirmationModal` or `ShareOptionsModal`.
- **Loading:** Boolean flag `isLoading` for skeleton rendering during initial fetch.
- **Error:** String/Object `errorState` to trigger ErrorBoundary or Toast.

---

## 8. API Communication & Security

Strict adherence to `.rules/api-communication.md` and `.rules/security.md`.

### API Integration
- Handled entirely within `src/features/itinerary/services/itineraryService.js`.
- Driven by `axiosInstance` which securely manages JWT injection via interceptors.
- **Page Component Role:** Uses a customized hook (e.g., `useItineraryFetch`) to execute the service on mount and populate local state. 

### Data Sanitization
- Although data is sourced from an internal API, any text rendered dynamically (e.g., special booking notes) must be processed through `sanitizeHTML` via the security utility layer to prevent XSS vulnerabilities.

---

## 9. UX Behavior & Interactions

Strict adherence to `.rules/ux-governance.md`.

### Loading States
- Renders **Skeleton UI** components that accurately match the structural silhouette of the Cards. 
- Avoids generic circular spinners for the initial page load to preserve layout stability in desktop mode.
- Button interactions (e.g. "Download Ticket") trigger localized inline circular loaders inside the button.

### Error Handling & Empty States
- **Fetch Failure:** Displays a centralized EmptyState/Error component suggesting a page refresh or contacting support.
- **Missing Optional Data:** If a flight has no layover, the layover divider and duration visualizer collapses automatically without breaking layout. Meal preferences collapse if unmapped.

### Accessibility (A11y)
- **Contrast:** Ensures text-primary against surface-card passes WCAG 2.1 AA.
- **Aria Labels:** Action buttons ("Download", "Print") must carry descriptive `aria-label`s.
- **Semantic HTML:** Page titles use `<h1/>`, sections use `<h2/>` / `<h3/>` for screen-reader continuity.

---

## 10. Performance & Production Hardening

Strict adherence to `.rules/performance.md`.

- **Memoization:** Read-only presentational cards (`FareBreakdownCard`, `BaggageRulesCard`) must be wrapped in `React.memo` as their props will not change post-fetch.
- **Lazy Loading:** `FlightItineraryPage` is lazy-loaded at the routing layer so the itinerary bundle is only downloaded by users navigating to it.
- **Utility Optimization:** Use shared pre-compiled formatting utilities for dates and currency to prevent unnecessary localized recalculations.

***
**END OF SPECIFICATION**
