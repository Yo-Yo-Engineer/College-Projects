# Accessibility Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve accessibility for [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Target WCAG 2.1 Level AA conformance as a baseline
- Prioritize issues that block access for users with disabilities

## Focus Areas

### Semantic HTML and Document Structure

- Verify semantic elements are used correctly: header, nav, main, section, article, aside, footer
- Ensure heading hierarchy is logical and sequential (h1 → h2 → h3, no skipped levels)
- Check for proper use of landmark regions for screen reader navigation
- Verify lists use ol, ul, dl — not divs styled as lists
- Ensure forms use proper label, fieldset, and legend elements

### ARIA and Screen Reader Support

- Verify ARIA roles, states, and properties are used correctly and only when semantic HTML is insufficient
- Check for aria-label, aria-labelledby, and aria-describedby on interactive elements that lack visible labels
- Ensure dynamic content updates use aria-live regions with appropriate politeness levels
- Verify custom components (modals, tabs, accordions, dropdowns) implement the correct ARIA pattern
- Check that ARIA attributes are not redundant with native HTML semantics

### Keyboard Navigation

- Verify all interactive elements are reachable and operable via keyboard (Tab, Shift+Tab, Enter, Space, Escape, Arrow keys)
- Ensure focus order follows a logical reading sequence
- Check for visible focus indicators on all interactive elements — do not remove outline without replacement
- Verify focus management for dynamic content: modals trap focus, removed elements return focus
- Ensure no keyboard traps exist — users can always navigate away from any element
- Check for skip-to-content links for bypassing repeated navigation

### Color and Visual Design

- Verify text-to-background color contrast meets WCAG AA: 4.5:1 for normal text, 3:1 for large text
- Ensure non-text contrast ratio is at least 3:1 for UI components and graphical objects
- Check that color is not the only means of conveying information (e.g., error states, status indicators)
- Verify content is readable and functional at 200% zoom
- Ensure text resizing up to 200% does not cause content loss or overlap

### Images and Media

- Verify all informative images have meaningful alt text describing purpose, not just appearance
- Ensure decorative images use alt="" or are applied via CSS
- Check for alternative text on icons used as interactive elements
- Verify complex images (charts, diagrams) have extended descriptions
- Ensure video has captions and audio has transcripts where applicable

### Forms and Interactive Elements

- Verify all form inputs have associated, visible labels (not just placeholder text)
- Ensure required fields are indicated visually and programmatically (aria-required)
- Check that error messages are associated with their fields and announced to screen readers
- Verify form validation errors are descriptive, specific, and suggest corrections
- Ensure autocomplete attributes are used for common fields (name, email, address)

### Motion and Animation

- Verify animations and transitions respect prefers-reduced-motion media query
- Ensure no content flashes more than three times per second (seizure risk)
- Check that auto-playing content can be paused, stopped, or hidden
- Verify carousels and sliding content have pause controls and are keyboard navigable

### Responsive and Adaptive Design

- Verify content reflows and remains usable at 320px viewport width (mobile)
- Ensure touch targets are at least 44×44 CSS pixels for mobile
- Check that content and functionality are equivalent across input modalities (touch, mouse, keyboard)

## Reference Standards

- WCAG 2.1 Level AA (Web Content Accessibility Guidelines)
- WAI-ARIA 1.2 (Accessible Rich Internet Applications)
- ARIA Authoring Practices Guide (APG) for component patterns
- Section 508 (U.S.) and EN 301 549 (EU) where applicable

## Constraints

- Preserve existing visual design and user experience where possible
- Prefer native HTML elements and built-in accessibility over ARIA workarounds
- Use ARIA attributes only when they complement (not conflict with) native semantics
- Ensure accessibility improvements do not break existing functionality

## Output

1. Accessibility issues identified, categorized by WCAG success criterion and severity
2. Changes made or proposed with impacted user groups noted
3. Keyboard navigation and screen reader test results
4. Color contrast audit results for affected components
5. Recommendations for automated and manual accessibility testing
6. Priority guidance: critical blockers, high-impact improvements, progressive enhancements
