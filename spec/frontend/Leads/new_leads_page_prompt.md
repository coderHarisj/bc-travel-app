# New Lead Page – Frontend Technical Specification

---

## Dependency

This document **extends**:
➡️ COMMON_FRONTEND_SPEC.md

---

## 1. Page Purpose

The New Lead page allows internal users to:
- Create a new tourism enquiry
- Capture customer travel intent
- Submit validated data securely

---

## 2. UI Composition

### Form Fields
- Name
- Mobile Number
- Email ID
- Travel Date
- Duration
- Destination
- Travel Type
- No. of Travelers
- Budget
- Lead Source
- Notes

---

## 3. Form Design Strategy

- Controlled inputs
- Field-level validation
- Required field enforcement
- Clear inline error messaging

---

## 4. State Management

- Form state remains local
- Redux not required for form data

---

## 5. Validation Rules

- Email format validation
- Mobile number length validation
- Mandatory field checks
- Date consistency validation

---

## 6. API Interaction

### POST /leads
- JWT protected
- Encrypted payload
- Normalized request structure
- Success & failure handling

---

## 7. Security Considerations

- Input sanitization
- Safe handling of notes field
- No sensitive data persisted on failure

---

## 8. UX Behavior

- Disable submit during API call
- Success redirect to View Leads page
- Cancel returns without side effects

---

