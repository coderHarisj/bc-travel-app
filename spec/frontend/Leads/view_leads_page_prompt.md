
---

# 📁 FILE 2 — `VIEW_LEADS_PAGE_SPEC.md`

```md
# View Leads Page – Frontend Technical Specification

---

## Dependency

This document **extends**:
➡️ COMMON_FRONTEND_SPEC.md

All global architecture, security, and authorization rules apply.

---

## 1. Page Purpose

The View Leads page enables internal users to:
- View all leads
- Search and filter lead data
- Paginate large datasets
- Perform lead-level actions

---

## 2. UI Composition

### Components
- Search Bar (Name / Email / Mobile)
- Date Range Filter (From – To)
- Leads Data Table
- Pagination Controls
- Row Action Menu

---

## 3. Data Table Specification

### Columns
- Lead Name
- Contact Info (Email + Mobile)
- Destination
- Travel Type
- Date & Time
- No. of Travelers
- Actions (3-dot menu)

---

### Capabilities
- Row selection
- Expandable rows (future-ready)
- Server-side pagination
- Row actions:
  - View
  - Edit
  - Convert to Quote
  - Delete

---

## 4. State Management

### Local State
- Search input
- Date filters
- Expanded row state
- Action menu state

### Redux State
- Lead list
- Pagination metadata
- Loading & error states

---

## 5. API Interaction

### GET /leads
- Params:
  - page
  - limit
  - searchText
  - fromDate
  - toDate
- JWT protected
- Encrypted request & response

---

## 6. Performance Considerations

- Debounced search input
- Memoized table rows
- Avoid unnecessary re-renders

---

## 7. Error Handling

- Inline empty states
- Graceful error messages
- Retry support

---

## END OF VIEW LEADS PAGE SPEC
