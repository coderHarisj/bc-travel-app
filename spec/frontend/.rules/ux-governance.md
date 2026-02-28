# UX Governance Rules
Project: Tourism Management System

## 1. Screen & Wireframe Clarity

For every screen:

- All fields must have clear labels.
- Mandatory fields must be visually indicated.
- Optional fields must be explicitly marked.
- Navigation flow must be consistent and predictable.
- No ambiguous UI elements.

If any field behavior is unclear from design:
→ Ask user before implementation.

---

## 2. Field-Level Validation

Each form must define:

- Required validations
- Format validations (email, phone, etc.)
- Length constraints
- Password strength rules (if applicable)

Validation must be:
- Inline
- Immediate (on blur or controlled submit)
- Human-readable

Never display technical validation errors.

Example:
Invalid email → “Please enter a valid email address.”
Weak password → Show password policy guidance.

---

## 3. Error Handling UX

All API errors must:

- Show user-friendly messages
- Avoid exposing system details
- Provide recovery guidance

System failure example:
“Something went wrong. Please try again.”

Never show:
- Stack traces
- Internal error codes
- Raw backend messages

---

## 4. Responsive Behavior

Responsive support is mandatory:

- Mobile (xs)
- Tablet (sm/md)
- Desktop (lg+)

Breakpoints must use MUI system only.
No custom media queries unless necessary.

Layout behavior must be defined:
- Forms stack vertically on mobile
- Tables scroll horizontally on xs
- Sidebar collapses on small screens

---

## 5. Accessibility Standards

Minimum: WCAG 2.1 AA

Requirements:
- Proper label association for inputs
- aria-* attributes where needed
- Keyboard navigation support
- Focus states visible
- Color contrast compliant
- Accessible validation messages

Forms must:
- Announce errors to screen readers
- Support tab navigation fully