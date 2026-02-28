# Frontend Technical Specification: Itinerary - Lodging Management

> **Version:** 1.0
> **Date:** 2026-02-28
> **Status:** DRAFT
> **Target Platform:** Desktop-Focused Web Application
> **Frontend Stack:** React, Material UI (MUI)

## 1. Executive Summary

This document outlines the frontend technical specifications for the "Lodging" event management feature within the Itinerary builder module of the Travel App. Based on the provided screenshots (`Itinerary-1.png`, `Itinerary-2.png`, `Itinerary-3.png`), this feature allows users to view a timeline of days, search an "Event Library" for lodging options, add a lodging block to a specific day, and populate or edit details such as hotel name, location, amenities, and check-in/check-out dates.

---

## 2. Component Reuse & Impact Analysis

Based on the project's React and MUI architecture, the following component structure is proposed. New components will strictly adhere to the project's atomic design and component guidelines.

### 2.1 Existing/Base Components (MUI & Custom Wrappers)
- `Typography`, `Box`, `Stack`, `Grid`: Core MUI layout and text components.
- `Button`, `IconButton`, `Fab`: For actions like "Save", "Preview", "Download", "Smart Import", and FABs.
- `Tabs`, `Tab`: For navigating between Event Library categories (Lodging, Flight, Transport, etc.).
- `TextField` / `Input`: For the search bar and inline editing.
- `Chip`: For tags/amenities (e.g., "Pool", "Parking", "Breakfast", "Wi-Fi").
- `DatePicker`: For Check-in/Check-out selection.
- `Avatar` / `Image`: For lodging thumbnails and large images.
- `Card`: For the Lodging options in the Event Library.

### 2.2 New Macro Components to Implement
- `ItineraryLayout`: The shell containing the top header, the main timeline area, and the right-side Event Library drawer/panel.
- `DaysTimeline`: The horizontal scrollable list of days (e.g., "Day 1 - Aug, 29", "Day 2 - Aug, 30") and "New Day" button.
- `DayEventBlankState`: Empty state for the "Events" section with a "Smart Import" button.
- `EventLibraryPanel`: The right-side panel containing category tabs, search, and a list of available resources.
- `LodgingLibraryCard`: Specific card implementation for a lodging item in the library, featuring an image, title, location, tags, and action buttons.
- `LodgingEventBlock`: The main block added to the timeline, with two states:
  - **Edit/Empty State:** Placeholders for "Add Image", "Enter Hotel Name", etc., and date pickers.
  - **Populated State:** Filled display with actual image, title, location string, and amenities chips.

---

## 3. Screen Overview & Desktop Layout Structure

The layout follows a standard desktop dashboard structure using MUI's Grid or Flexbox (`Stack`).

### 3.1 Global Layout
- **Top Navbar:** Contains breadcrumbs/title ("Itinerary"), actionable text buttons ("Preview", "Download", "Create Virtual Tour"), and a solid "Save" primary button.
- **Hero Image Section:** A wide banner for the destination (e.g., "Kashmir") with an edit icon and a dropdown for audience type ("Ideal For: Adults").
- **Main Content Area:** Split into two primary columns below the hero banner.
  - **Left Column (approx. 70% width):** The `DaysTimeline` and the `Events` list for the selected day.
  - **Right Column (approx. 30% width):** The `EventLibraryPanel` fixed to the side or within a grid column.

### 3.2 Main Content Breakdown
- **DaysTimeline:** A horizontally scrollable row of cards representing each day. The active day has a distinct solid background, while inactive days have a white background with a border.
- **Events Section (Left Column):**
  - Header: "Events".
  - Empty State: Pure white canvas with a "Smart Import" button at the top right.
  - Filled State: Contains `LodgingEventBlock` cards stacked vertically.
- **Event Library (Right Column):**
  - Section Header: "Event Library".
  - Search Input: Full width, with a search icon and filter icon adjecent.
  - Category Tabs: A grid of selectable buttons (Lodging, Flight, Transport, etc.). "Lodging" is active with a solid color.
  - Results List: Vertically scrollable list of `LodgingLibraryCard` components.

---

## 4. Design System Extraction

The UI will be built using a customized MUI Theme. No hardcoded colors or magic pixel values will be used.

### 4.1 Color Palette (Approximate extractions to map to Theme)
- **Primary:** Dark Teal/Green (used for active states, primary buttons, Hero overlay).
- **Secondary / Actions:** White for text on Primary, Dark Gray for standard text.
- **Backgrounds:**
  - App Background: Light gray/off-white (e.g., `#F5F7F9` or similar).
  - Cards/Panels: White (`#FFFFFF`).
  - Active Day / Library Tab: Teal/Green.
- **Chips/Tags:** Light gray background with dark text.

### 4.2 Typography Scale
- **Headers (h1, h2, h3):** Used for "Itinerary", "Kashmir", "Events", "Event Library". Semi-bold weight.
- **Body1/Subtitle1:** Used for Card Titles (e.g., "Kashmir Holiday Resort"). Medium weight.
- **Body2/Caption:** Used for locations, dates, and chip labels. Regular weight.

### 4.3 Spacing System
- Strict adherence to MUI's 8px grid system (`theme.spacing()`).
- High padding within cards (approx `p={2}` or `p={3}` for Event blocks).
- Gap between layout columns defined by `theme.spacing(3)` or `theme.spacing(4)`.

---

## 5. Field-Level Specification

### `LodgingEventBlock` (Edit/Add State)
| Field | UI Component | Validation & Rules |
| :--- | :--- | :--- |
| **Hero Image** | Upload Area / Placeholder | Click to upload. Accepts `.jpg`, `.png`. Aspect ratio ~16:9. |
| **Hotel Name** | `TextField` (variant="standard" or hidden borders) | Required. Max length 100 chars. |
| **Location** | `TextField` with leading Pin Icon | Required. Tied to Google Places autocomplete if applicable. |
| **Amenities** | Box with "Add Amenities" button | Opens a modal/popover to select predefined tags or type new ones. |
| **Check-in Date/Time** | `DatePicker` / `DateTimePicker` | Required. Must be chronologically before Check-out. Default aligns with the active day. |
| **Check-out Date/Time**| `DatePicker` / `DateTimePicker` | Required. Must be chronologically after Check-in. |
| **Remove Action** | `IconButton` (Trash/Close icon) | Removes the block from the timeline. Prompts confirmation if data exists. |

---

## 6. Interaction & UX Behavior

Adhering to `.rules/ux-governance.md`:

- **Active States:** The selected "Day" in the timeline and the selected "Category" in the Event Library must have a strong visual indicator (solid background fill and white text).
- **Empty States:** Clear calls to action. The Events empty state relies on "Smart Import" or presumably drag-and-drop/clicking from the Event Library. The Lodging block empty state provides inline ghost-text guiding the user ("Enter Hotel Name").
- **Drag & Drop (Assumed/Proposed):** Users should be able to drag a `LodgingLibraryCard` from the right panel into the left "Events" area to instantly populate a `LodgingEventBlock`.
- **Inline Editing:** When a `LodgingEventBlock` is added, fields should be editable inline (clicking "Enter Hotel Name" transforms it into an active input) rather than opening a separate bulky modal.
- **Feedback & Errors:** Inline validation for dates (e.g., Check-out before Check-in) must display human-readable text below the picker. "Please ensure check-out is after check-in." No system alerts.

---

## 7. Accessibility Specification (WCAG 2.1 AA)

- **Keyboard Navigation:** All interactable elements (Tabs, Cards, FABs, Inputs, DatePickers) must be reachable via `Tab` key. Focus rings must be visible.
- **ARIA Labels:**
  - Event Library tabs must have `role="tab"` and `aria-selected` attributes.
  - The "Smart Import" button needs an `aria-label`.
  - Icon buttons (like the `+` FAB or Remove block button) must have descriptive `aria-label`s (e.g., `aria-label="Add Kashmir Holiday Resort to itinerary"`).
- **Color Contrast:** The teal/green background with white text must pass contrast ratio tests. If the teal is too light, it must be darkened to meet the 4.5:1 text contrast requirement.
- **Screen Readers:** Empty state placeholders must be properly associated with inputs using `<label>` or `aria-labelledby`.

---

## 8. State Management Plan

Given the complex UI, state should be managed locally for transient UI changes and globally for the itinerary data.

### Global State (Context API or Redux/Zustand)
- `itineraryData`: The core object holding the destination, days, and arrays of events assigned to each day.
- `activeDayId`: Tracks which day is currently selected in the timeline.
- `eventLibraryCategory`: Tracks the active tab in the side panel.

### Local Component State
- `searchQuery`: Local to the `EventLibraryPanel` for filtering the list.
- `isEditing`: Boolean local to `LodgingEventBlock` to toggle between static view and input fields.
- `draggedItem`: For managing drag-and-drop UX states if implemented.

---

## 9. API Integration Requirements

- **GET `/api/v1/library/lodging`:** Fetch list of available hotels based on location (e.g., Kashmir). Supports search query params.
- **PATCH `/api/v1/itinerary/{id}/events`:** Save the newly added `LodgingEventBlock` to the specific day's array. Required payload: `eventId`, `dayId`, `type: 'lodging'`, `details: { hotelName, checkIn, checkOut, ... }`.
- **POST `/api/v1/upload`:** Endpoint to handle the "Add Image" action for custom lodging entries.

---

## 10. Performance Considerations

- **Virtualization:** If the Event Library search returns hundreds of results, implement `react-window` or MUI's virtualized list to maintain DOM performance.
- **Image Optimization:** Thumbnails in the Event Library and the Hero image must be served in next-gen formats (WebP) and correctly sized to prevent layout shift and slow load times. Ensure lazy-loading (`loading="lazy"`) for images below the fold in the Event Library.
- **Debouncing:** The search input in the Event Library must be debounced (e.g., `300ms`) to prevent rapid, unnecessary API calls while the user points.

---

## 11. Open Questions for Clarification

1. **Adding Events:** Does adding an event occur solely by picking/dragging from the "Event Library", or can the user click a blanket "Add Event" button in the main area to start from scratch?
2. **"Smart Import":** What is the exact workflow triggered by the "Smart Import" button? Does it parse a PDF/email or open an AI prompt?
3. **Data Saving:** Are changes auto-saved upon blurring a field in the inline editor, or only when the top right "Save" button is explicitly clicked?
