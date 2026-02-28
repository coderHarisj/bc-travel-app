# Technical Specification: Activity Itinerary

**Date:** 2024-10-24
**Target Path:** `e:\POC\bc-travel-app\spec\frontend\Itinerary\Activity\activity_itinerary_spec.md`
**Prepared By:** Senior Frontend Architect

## 1. Feature Overview
This specification details the frontend implementation for the "Activity Itinerary" feature. It covers both the **Consumer View** (a read-only presentation of planned activities for a day) and the **Builder View** (an editable interface for travel agents/users to add and configure activities within a trip).

The interface is completely desktop-focused. The Activity module must integrate seamlessly within the existing trip builder layout and adopt a clean, structured consumer presentation.

### Screens Covered
1. **Activity Itinerary (Consumer View)**: `Itinerary.png` - Standard list view of activities for a specific day.
2. **Activity Details (Expanded/Consumer)**: `Itinerary (1).png` - Detailed view of a specific activity with tabs for Inclusions, Policy, and Summary.
3. **Builder View (Empty Activity State)**: `Itinerary (2).png` - The itinerary builder interface showing an unconfigured "Add Event" placeholder for an activity.
4. **Builder View (Filled Activity State)**: `Itinerary (3).png` - The itinerary builder interface showing a populated activity card alongside the Event Library.

---

## 2. Component Ecosystem Analysis

### Reused Components
*   **Typography & Primitives:** Standard headings, body text, buttons (`Primary`, `Secondary`, `Ghost`), and dividers from the core Design System.
*   **`Tabs` Component:** Standard navigation tabs component (used in the Activity Details view).
*   **`Chip` / `Tag` Component:** Used for status indicators (e.g., "Non-refundable") and sub-navigation (e.g., "Sightseeing", "Food/Drink").
*   **`IconButton`:** Used across both views for actions like expanding details or removing an event.
*   **`Accordion` / `ExpandableSection`:** Core expanding mechanism used for showing full details in the consumer view.
*   **Builder Layout Shell:** The overarching layout of the Builder View (Top Image Cover, Itinerary Summary Bar, Day Stepper, Left Events Panel, Right Library Panel) is reused across all itinerary types (Flight, Activity, Transport, etc.).

### Extended Components
*   **`EventCard_Builder`:** Extended to support "Activity" specific data blocks vertically aligned (Image placeholder, Title block, Grid for configurable metrics: Booked Through, Confirmation, Carrier, Price).
*   **`EventLibraryCard`:** Extended to display activity-specific image, title, and location structure.

### New Components
*   **`ActivityCard_Consumer`:** A presentation-focused card specifically for viewing an assigned activity. It displays an image, title, brief description, location, time, and primary action ("View voucher").
*   **`ActivityDetailsPanel`:** The detailed expansion element for `ActivityCard_Consumer` separated into categorized tabs: "Details", "Inclusions & Exclusions", "Policy", and "Summary".
*   **`ActivityPriceSummaryPanel`:** A bottom-anchored summary section displaying the total price breakdown and support links ("Need help with this booking?").
*   **`ActivityLibraryFilter`:** The sub-navigation component inside the Event Library for activities containing "Sightseeing", "Food/Drink", and "Cruise".

---

## 3. Layout & Grid Specifications

### Consumer View (`Itinerary.png`, `Itinerary (1).png`)
*   **Layout Structure:** Centered single-column layout with a maximum constrained width (approx. `max-w-3xl` or `800px`) for optimal readability.
*   **Card Spacing:** Distinct vertical spacing (`gap-4` or `16px`) between individual activity cards.
*   **Internal Layout (`ActivityCard_Consumer`):**
    *   Horizontal composition.
    *   Left side: Image container (approx. `120px` width, square/rounded aspect).
    *   Right side: Content stack leveraging CSS Flexbox (`flex-col`), top-aligned elements (Title, Desc text), and vertically-centered meta rows (Location, Time) with gaps (`gap-2` or `8px`).

### Builder View (`Itinerary (2).png`, `Itinerary (3).png`)
*   **Global Grid:** 2-column asynchronous layout for the main content area below the common builder headers.
    *   **Main Events Column (Left):** Occupies approximately `65-70%` width.
    *   **Event Library Column (Right):** Occupies approximately `30-35%` width. Fixed right-side panel with independent scrolling.
*   **Internal Layout (`EventCard_Builder` for Activity):**
    *   **Empty State:** Gray placeholder background mimicking the expected final layout shape. Central "Add Image" icon.
    *   **Filled State:** 
        *   Horizontal Flex layout: Left (Image `150px` width), Right (Details Area).
        *   Details Area Grid: Header Row (Title), Sub-header Row (Description, Location, Timings), Footer Grid (4 columns for Booked Through, Confirmation, Carrier, Price).

---

## 4. Design System Integration

*   **Colors:**
    *   Primary text: Dark Slate / Almost Black (e.g., `#1A1A1A`).
    *   Secondary text: Gray (e.g., `#666666`).
    *   Backgrounds: Main layout background uses a soft off-white/gray, while cards use pure White (`#FFFFFF`).
    *   Action Colors: Teal/Brand Green for primary actions (e.g., "Smart Import" button, "View voucher" text link).
    *   Borders: Subtle light gray border standard (`1px solid #E5E5E5`) outlining cards and dividing sections.
*   **Typography:**
    *   Headers: Bold presentation (e.g., `font-weight: 700`), scaling from `h3` (titles) to `h6` (sub-section headers).
    *   Metadata: Small text sizes (`text-sm` or `12px/14px`) for timing, location, booking identifiers.
*   **Radii & Shadows:**
    *   Consistent border radius (`rounded-lg` or `12px`) for cards and inner imagery.
    *   Subtle drop shadows on active/hovered components within the Event Library, while Consumer cards seem flat with prominent borders.

---

## 5. Component Level Specifications

### `ActivityCard_Consumer`
*   **Purpose:** The read-only card for an activity displayed on the consumer timeline.
*   **Props:**
    *   `activityId`: string
    *   `title`: string
    *   `description`: string
    *   `imageUrl`: string
    *   `location`: string
    *   `timeRange`: string (e.g., "13:00 - 15:30")
    *   `isExpanded`: boolean (drives expansion state)
    *   `onExpandToggle`: function
*   **Behavior:** Clicking on the card body or explicit "expand" control triggers the reveal of the `ActivityDetailsPanel`.
*   **Styling:** Solid border. The "View voucher" acts as a primary ghost button aligned to the bottom right of the content grid.

### `ActivityDetailsPanel`
*   **Purpose:** Expanded details block nested within/under `ActivityCard_Consumer`.
*   **State:** Local state to manage active tab index (`Details` | `Inclusions & Exclusions` | `Policy` | `Summary`).
*   **Styling:** Wrapped in a subtle bordered container with a light background to distinguish from the primary card header.

### `EventCard_Builder` (Activity Variant)
*   **Purpose:** The editable card block in the trip building timeline.
*   **Props:**
    *   `mode`: 'empty' | 'filled'
    *   `type`: 'sightseeing' | string
    *   `data`: ActivityDataObj (null for empty mode)
    *   `onRemove`: function
*   **Behavior (Empty Mode):** Prompts user to click and input details manually or drag an item from the Event Library. Areas for "Add Event Name", "Add Location", etc., contain placeholder input styling.
*   **Behavior (Filled Mode):** Displays finalized data. The bottom row structures business metadata (`Booked Through`, `Confirmation`, `Carrier`, `Price`) into a tight 4-column responsive grid. Features an explicit "Remove" string-button top right.

### `ActivityLibraryFilter`
*   **Purpose:** Sub-category filtering in Event Library.
*   **Implementation:** Rendered as a horizontal scrolling or wrapping flex container of `Tabs` (Sightseeing, Food/Drink, Cruise) displaying under the primary category selection. Active tab indicated by bold text and an underscore brand-color indicator.

---

## 6. Content & Field Specifications

### Consumer View Fields
| Element | Rules/Formatting |
| :--- | :--- |
| **Itinerary Title** | Main header format: "[Location] itinerary" (e.g., "Activity itinerary"). |
| **Subtitle** | Format: "Trip to [Destination]". |
| **Activity Title** | Truncate if spans more than 2 lines. Heavy font weight. |
| **Description** | Brief snippet. Max 2 lines. Ends with ellipsis if overflowing. |
| **Location Format** | "[City/Area]" + Icon prefix. |
| **Time Format** | "[HH:MM] - [HH:MM]". 24h or localized 12h format. |

### Builder View Additional Fields
| Element | Rules/Formatting | Type | Required? |
| :--- | :--- | :--- | :--- |
| **Booked Through** | Text field for agency or platform (e.g., "GT holidays") | String | No |
| **Confirmation** | Alphanumeric booking reference. | String | No |
| **Carrier** | Operator name (e.g., "GT holidays"). | String | No |
| **Price** | Numeric format, optionally prefixed with currency (e.g., "2000"). Allow inline editing. | Number | No |
| **Event Name** | Input text field. High contrast placeholder when empty. | String | **Yes** |

---

## 7. Actions & State Management

*   **Consumer Details Expansion:** Toggling an activity displays a panel rendering contextually rich tabs. A secondary API call should not be required if payload pre-delivered; use local UI state (`isExpanded`, `activeTabId`).
*   **Builder Event Removal:** Clicking "Remove" removes the element from the global `Events` array. Needs a confirmation transient state to prevent accidental drops.
*   **Drag & Drop state (Implied via Builder):** Interacting with the "Event Library" implies the user can select an item (e.g., "Gondola Cable Car Ride") which subsequently transitions the `EventCard_Builder` from 'empty' to 'filled' state.
*   **Library Filtering:** Selection of "Sightseeing", "Food/Drink", "Cruise" refilters the array driving the right-column list component.

---

## 8. Accessibility (A11y)

*   **ARIA Roles:** Ensure expansion regions (the consumer `ActivityDetailsPanel`) utilize `aria-expanded` and `aria-controls`.
*   **Focus Management:** 
    *   Tabbing between tabs within details should work natively (left/right arrow navigation if utilizing ARIA tab patterns).
    *   The "Remove" action inside the builder requires clear `aria-label` attribute specifically defining *what* is being removed to avoid context loss on screen readers.
*   **Empty State Clarity:** Screen readers should announce "Empty Activity slot ready to add event name" for `EventCard_Builder` in 'empty' mode.

---

## 9. Responsive Behavior

*   **Constraint:** This specification is explicitly DESKTOP FOCUSED.
*   **Graceful Resizing (within desktop):**
    *   Consumer view main column should remain centered and max-width clamped across larger horizontal viewports.
    *   Builder left/right split ratios (approx 70/30) should be defined by `flex-basis` or CSS grid fractions (`fr`), allowing fluid resize down to standard `1024px` widths. Below `1024px` is out of scope for this desktop spec, but standard grid wrap logic would typically apply.
    *   Business metadata row in `EventCard_Builder` (Booked Through, etc.) should allow flex-wrapping if data length exceeds column width.

---

## 10. Dependencies & Integrations

*   **Icons Library:** Usage of `Icon` component mapping to mapping/timing/ticket SVGs.
*   **Builder Context:** Deeply integrated with the overarching Builder context (State management representing "Trip", "Selected Day", "Day Events Array").
*   **Event Library Data Service:** Relies on an endpoint specifically returning filtered lists of 'Activities' mapping to categories (Sightseeing, etc.) with coordinates/details.
