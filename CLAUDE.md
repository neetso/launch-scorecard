# launch-scorecard - Claude Code Instructions

## Project Overview

[Brief description - fill in as the project evolves]

## Browser Testing

Use Chrome to verify UI changes. Don't assume code works - test it.

### When to test
- After any visual or layout changes
- After adding/modifying interactive elements (buttons, forms, navigation)
- After changing state management or data flow that affects the UI
- Before considering a UI-related task complete

### When you don't need to test
- Pure backend/API changes with no UI impact
- Config file changes
- Documentation updates
- Refactors that don't change behavior (though a quick sanity check doesn't hurt)

### What to verify
- The change works as expected
- Browser console is free of new errors/warnings
- Mobile viewport (< 768px) if UI is affected
- Edge cases: empty states, long text, error states

## Development

**Dev server:** [TBD - e.g., `npm run dev`]

**URL:** [TBD - e.g., `http://localhost:5173`]
