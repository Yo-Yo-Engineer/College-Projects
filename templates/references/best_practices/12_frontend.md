# Frontend Development Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve the frontend implementation of [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Review component design, state management, performance, and user experience
- Justify any new library or framework additions before introducing them

## Focus Areas

### Component Architecture and Design

- Verify components follow single responsibility — each component has one clear purpose
- Ensure a clear separation between presentational (UI) and container (logic/data) components
- Check for appropriate component granularity — not too large (god components) or too small (over-fragmented)
- Verify component composition is preferred over deep inheritance or prop drilling
- Ensure reusable components are generic, well-documented, and free of business logic
- Check for consistent file and folder organization matching the project's adopted conventions

### State Management

- Verify state is owned at the appropriate level — local state for component-specific data, shared state for cross-component data
- Ensure server state (API data) and client state (UI state) are managed separately where practical
- Check for unnecessary global state — keep state as local as possible
- Verify state updates are immutable and predictable
- Ensure derived state is computed, not duplicated and manually synchronized
- Check for stale state issues, especially with async operations and closures

### Data Fetching and API Integration

- Verify API calls are centralized through a service layer or data-fetching hooks — not scattered across components
- Ensure loading, error, and empty states are handled for all async operations
- Check for proper request cancellation when components unmount or inputs change
- Verify caching and request deduplication for repeated data fetches
- Ensure sensitive data (tokens, credentials) is not exposed in client-side code or browser storage
- Check for proper error handling and user-facing error messages on API failures

### Forms and Validation

- Verify form validation happens both client-side (UX) and server-side (security)
- Ensure validation rules are consistent between frontend and backend
- Check for accessible form patterns: labels, error association, required field indication
- Verify form state management handles dirty tracking, submission states, and optimistic updates
- Ensure user input is sanitized to prevent XSS — avoid dangerouslySetInnerHTML or equivalent without sanitization

### Routing and Navigation

- Verify routes are organized, named consistently, and match the URL structure conventions
- Ensure protected routes enforce authentication and authorization checks
- Check for proper handling of 404, unauthorized, and error routes
- Verify deep linking and browser navigation (back/forward) work correctly
- Ensure route-level code splitting is configured for large applications

### Performance and Bundle Optimization

- Verify code splitting and lazy loading for routes and heavy components
- Check for tree shaking effectiveness — no unused code bundled in production
- Ensure images are optimized: proper formats, responsive sizes, lazy loading
- Verify memoization is used appropriately for expensive computations and stable references — not over-applied
- Check for unnecessary re-renders caused by unstable references, inline functions, or missing keys
- Ensure third-party library usage is justified — check bundle impact before adding dependencies

### Error Handling and Resilience

- Verify error boundaries catch and display errors gracefully — not white screens
- Ensure global error handling captures unhandled exceptions and promise rejections
- Check for proper fallback UI for failed components and network errors
- Verify retry mechanisms for transient failures (network drops, API timeouts)
- Ensure errors are reported to an error tracking service with sufficient context

### Styling and Layout

- Verify a consistent styling approach is used across the project (CSS modules, CSS-in-JS, utility-first, or preprocessor)
- Ensure responsive design works across target breakpoints and devices
- Check for consistent use of a design system or theme (spacing, typography, color tokens)
- Verify no inline styles for layout or theming — use the project's styling conventions
- Ensure dark mode and theme switching work correctly if supported

### Security

- Verify no sensitive data is stored in localStorage, sessionStorage, or cookies without appropriate protection
- Ensure all user input displayed in the UI is sanitized to prevent XSS
- Check that authentication tokens use secure, httpOnly cookies or equivalent secure storage
- Verify CORS and CSP policies are configured correctly
- Ensure third-party scripts and resources use Subresource Integrity (SRI) where applicable

### Testing

- Verify components have unit tests for logic and behavior — not just snapshot tests
- Ensure integration tests cover key user flows and form interactions
- Check for proper use of testing utilities: render, user events, async waiters
- Verify accessibility-focused tests (role queries, ARIA assertions)
- Ensure visual regression testing or design review process exists for UI changes

## Reference Standards

- Component-based architecture principles
- Responsive Web Design (mobile-first approach)
- Core Web Vitals (LCP, FID/INP, CLS)
- OWASP Top 10 client-side security risks
- Progressive Enhancement

## Constraints

- Preserve existing component patterns and styling conventions
- Justify any new state management libraries or styling approaches before introducing them
- Prefer standard platform APIs over third-party polyfills where browser support is sufficient
- Ensure all changes are visually consistent with the existing design system

## Output

1. Component architecture and design issues identified
2. State management improvements made or proposed
3. Performance optimizations with measurable impact
4. Security and accessibility issues resolved
5. Testing gaps identified and addressed
6. Recommendations for build optimization, tooling, or follow-up work
