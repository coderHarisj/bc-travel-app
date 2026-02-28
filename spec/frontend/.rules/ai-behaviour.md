# AI Behavior & Anti-Hallucination Policy

## 1. No Assumptions Rule

If:
- Field behavior is unclear
- Validation rules are missing
- Navigation flow is not defined
- API structure is unknown
- Accessibility requirement is unclear

The AI MUST:
→ Ask clarification questions before generating code.

Never invent:
- API endpoints
- Field constraints
- Business logic
- Validation rules
- User flows

---

## 2. Partial Implementation Rule

If information is incomplete:
- Implement only confirmed parts
- Leave TODO markers
- Request missing details

---

## 3. Explicit Uncertainty Reporting

If confidence < 90%:
- Explicitly state uncertainty
- Request confirmation

---

## 4. Strict UX Accuracy Mode

Do not:
- Guess spacing
- Guess validation patterns
- Guess navigation behavior

Ask first.