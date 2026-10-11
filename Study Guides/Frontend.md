# Frontend Architecture and Design Systems: A Beginner's Study Guide

This is an original, plain-language guide to frontend architecture and design systems. It covers general ideas for learning and practice; it is not a chapter-by-chapter summary or a reproduction of *Frontend Architecture for Design Systems*.

## Who this guide is for

This guide is for people who build websites or web applications and want to make their frontend code easier to understand, reuse, and change. You can use the ideas with React, Vue, Angular, or plain HTML, CSS, and JavaScript.

## Table of contents

1. [The big picture](#1-the-big-picture)
2. [What frontend architecture means](#2-what-frontend-architecture-means)
3. [Choosing good component boundaries](#3-choosing-good-component-boundaries)
4. [What a design system is](#4-what-a-design-system-is)
5. [Design tokens and visual consistency](#5-design-tokens-and-visual-consistency)
6. [Organizing styles](#6-organizing-styles)
7. [Building reusable components](#7-building-reusable-components)
8. [Working with a design system](#8-working-with-a-design-system)
9. [Testing, accessibility, and quality](#9-testing-accessibility-and-quality)
10. [Growing and maintaining the system](#10-growing-and-maintaining-the-system)
11. [A practical example](#11-a-practical-example)
12. [Study exercises](#12-study-exercises)
13. [Quick glossary](#13-quick-glossary)
14. [Project checklist](#14-project-checklist)
15. [Architecture patterns and trade-offs](#15-architecture-patterns-and-trade-offs)
16. [Data, state, and service boundaries](#16-data-state-and-service-boundaries)
17. [Performance, resilience, and responsive behavior](#17-performance-resilience-and-responsive-behavior)
18. [Design-system delivery and versioning](#18-design-system-delivery-and-versioning)
19. [Common problems and how to avoid them](#19-common-problems-and-how-to-avoid-them)
20. [Learning resources](#20-learning-resources)

## 1. The big picture

A frontend is the part of an application people see and use in a browser. It includes the page structure, visual styling, interactions, and the code that connects the interface to data.

As an application grows, small choices become important. If every page creates its own button, colors drift. If one component handles too many jobs, changing it becomes risky. Frontend architecture helps organize the work so a team can build new features without making the whole application harder to maintain.

A useful mental model has three parts:

| Part | What it contains | Example |
| --- | --- | --- |
| Product features | Screens and workflows for users | Search page, account settings |
| Shared interface | Reusable visual and interaction patterns | Button, form field, navigation |
| Foundations | Rules and tools that support both | Colors, spacing, build setup, accessibility rules |

The goal is not to create the most layers or the most reusable code. The goal is to make the next change understandable and safe.

## 2. What frontend architecture means

Frontend architecture is the way code and responsibilities are arranged in an application. Good architecture makes it easier to answer questions such as:

- Where should this feature live?
- Which components can be reused?
- Which code is allowed to depend on which other code?
- How can we change a shared style without breaking many pages?

### Organize around responsibilities

An application might separate:

- **Pages or routes:** Put together the parts needed for a particular screen.
- **Features:** Implement user tasks, such as signing in or editing a profile.
- **Shared components:** Provide common interface pieces used in several places.
- **Foundations:** Hold design values, utilities, and application-wide setup.

These are useful categories, not rules that every project must follow. A small application may need only a few folders. A large application may need clearer boundaries. Start with the simplest structure that helps the team find and change code.

### Keep dependencies understandable

When one part of the application uses another, it depends on it. A page depending on a shared button is usually sensible. A low-level button importing a specific account page is a warning sign: shared code now knows about a particular feature.

Prefer dependencies that flow from specific features toward general tools:

```text
Page -> Feature -> Shared component -> Foundation
```

Avoid circular dependencies, where A needs B and B also needs A. They make code harder to test and change.

### Choose consistency over cleverness

Use patterns people on the team can recognize. A clear component with a few well-named properties is often better than a highly configurable component with many hidden behaviors.

## 3. Choosing good component boundaries

A component is a piece of the interface with a defined job. It may show content, manage a small interaction, or combine other components.

Make a component when it:

- Appears in more than one place.
- Has a clear responsibility or interaction.
- Needs to be tested or changed independently.
- Makes a complicated screen easier to read.

Do not split every line of markup into its own component. Too many tiny pieces can make it difficult to see how a screen works.

### Give each component a clear job

A `SearchField` might display an input and a clear button. It should not also fetch every kind of application data, decide which page is visible, and manage unrelated settings.

Ask:

1. What does this component display or do?
2. What information does it need from its parent?
3. What user actions does it report back?
4. Can its name describe its purpose?

### Use clear inputs and outputs

Inputs (often called props or parameters) tell a component what to display. Outputs (often callbacks or events) tell the parent that something happened.

For example, a save button might receive a label and a disabled state, then report when it is clicked. It should not need to know how the entire application stores data.

### Reuse patterns, not accidental similarities

Two things that look alike may have different behavior. Two things that look different may share the same interaction rules. Reuse is most helpful when the parts have a stable shared purpose, not just a few matching lines of code.

## 4. What a design system is

A design system is a shared way to design and build an application's interface. It usually includes:

- **Design principles:** The qualities the interface should have, such as being clear and accessible.
- **Design decisions:** Colors, type styles, spacing, and other visual rules.
- **Components and patterns:** Reusable pieces such as buttons, alerts, and navigation.
- **Guidance:** Examples of when and how to use those pieces.
- **Tools and ownership:** The code, documentation, and team practices that keep it useful.

A component library is one part of a design system. A folder of components without shared rules or usage guidance is not necessarily a complete design system.

### Why teams use one

A useful system can:

- Make common tasks faster to build.
- Give related pages a consistent feel.
- Help teams apply accessibility improvements in shared components.
- Make product-wide changes easier to coordinate.

It also has costs. Shared components need maintenance, documentation, and a way to handle different product needs. Build only what has a real user or team benefit.

## 5. Design tokens and visual consistency

Design tokens are named values for design decisions. Instead of repeating a color code throughout the application, code can refer to a name such as `color-text-primary`.

```css
:root {
  --color-text-primary: #202124;
  --color-surface: #ffffff;
  --color-action: #155eef;
  --space-small: 0.5rem;
  --space-medium: 1rem;
  --radius-control: 0.375rem;
}
```

Then a component can use the names:

```css
.button {
  color: var(--color-surface);
  background: var(--color-action);
  border-radius: var(--radius-control);
  padding: var(--space-small) var(--space-medium);
}
```

If the approved action color changes, updating the token can update every component that uses it.

### Keep names about purpose

Names such as `color-action` or `space-medium` explain the role of a value. Names such as `blue-7` or `margin-12` describe its current appearance or measurement, but not why it exists.

Some systems use multiple token layers:

1. **Base values:** Raw colors and sizes.
2. **Meaningful tokens:** Names based on purpose, such as `color-border`.
3. **Component tokens:** Optional values for a specific component, such as `button-background`.

Do not add layers unless they help people make or update decisions. A small system may need only meaningful tokens.

### Support themes deliberately

Themes, such as light and dark modes, can provide different values for the same meaningful tokens. Components can continue using `color-surface` while a theme changes what that token means.

Check contrast, focus indicators, and disabled states in every theme. Changing colors alone does not guarantee an accessible theme.

## 6. Organizing styles

Styles should be predictable and limited in scope. A change to one feature should not unexpectedly restyle unrelated parts of the application.

Helpful practices include:

- Prefer component-scoped styles, CSS Modules, or another clear scoping approach.
- Use shared tokens for repeated design decisions.
- Use global styles sparingly for true application-wide rules, such as base typography.
- Avoid selectors that depend on a long chain of page-specific markup.
- Use names that describe the component or purpose.
- Check responsive layouts at narrow and wide screen sizes.

Do not treat a CSS naming method or styling library as a replacement for good boundaries. The important thing is that the team can find where a rule comes from and understand what it affects.

## 7. Building reusable components

A reusable component should be easy to understand and difficult to misuse.

### Start with the real use cases

Before making a component generic, list where it will be used. For example, an alert might need:

- A message.
- A severity such as information, warning, or error.
- An optional action.
- A way to dismiss it, if appropriate.

Build for needs that exist or are likely, not every option someone might imagine.

### Prefer a small, clear API

A component API is the set of choices callers can make. Too many options make it hard to know which combinations are valid.

Prefer a few meaningful properties over:

- Many boolean flags that interact in surprising ways.
- A generic `config` object with undocumented fields.
- Internal knowledge of the pages where the component is used.

Use sensible defaults, but make important behavior visible to the caller. Document any constraints.

### Keep state close to where it is used

State is information that can change, such as whether a menu is open. Keep it near the component that owns the behavior unless another part of the application needs to control it. Moving all state to a global store too early can make simple behavior harder to follow.

## 8. Working with a design system

A design system works best when product teams can find, understand, and use it.

### Show components in context

Documentation should explain:

- What the component is for.
- When to use it and when not to.
- Its available states and options.
- Examples of correct use.
- Accessibility behavior and known limitations.

Pictures are helpful, but working examples and plain-language guidance are often more useful.

### Make the system easy to adopt

Publish components in a way that fits the application's build tools. Explain how to install or import them. Provide examples that match real product code. If teams must work around the system to complete common tasks, investigate why.

### Decide how changes are managed

Agree on who reviews shared components, how breaking changes are announced, and how teams can request improvements. A contribution process should be clear without making every small change difficult.

## 9. Testing, accessibility, and quality

Shared components can affect many screens, so test both their behavior and their appearance in important contexts.

### Useful kinds of checks

- **Unit or component tests:** Check behavior such as opening a menu or reporting a selection.
- **Integration tests:** Check that related pieces work together on a page.
- **Visual checks:** Compare rendered screens to catch unexpected layout or style changes.
- **Accessibility checks:** Combine automated tools with keyboard and assistive-technology testing.
- **Build checks:** Confirm that the application can compile and that package boundaries work.

No single test proves a component is correct. Choose checks based on the risks and the ways people use the feature.

### Treat accessibility as part of the design

Accessible behavior should be considered while designing and building, not added at the end. Check that:

- Controls have meaningful names and appropriate semantics.
- Keyboard users can reach and operate interactive elements.
- Focus is visible and moves in a sensible order.
- Text and important graphics have sufficient contrast.
- Errors are explained in a way users can identify and correct.
- Motion can be limited when users request reduced motion.

Prefer native HTML controls when they fit the need. They already provide useful browser and assistive-technology behavior.

## 10. Growing and maintaining the system

A design system is a product that changes over time. It needs attention just like the applications that use it.

### Add things in response to real needs

When a team asks for a new component, first understand the user problem. Check whether an existing component can solve it. If several teams have the same need, a shared addition may be worthwhile.

### Change shared components carefully

Before changing a shared component:

1. Find out where it is used.
2. Decide whether the change is compatible with current uses.
3. Add or update tests and examples.
4. Explain any migration steps.
5. Communicate changes to the teams affected.

Avoid adding an option for every one-off difference. Too many special cases can turn a simple shared component into a difficult-to-maintain framework.

### Measure whether it helps

Useful signals include whether teams can find components, whether they reuse them, how often they need workarounds, and whether accessibility or consistency improves. Adoption numbers alone do not show that a system is useful.

## 11. A practical example

Imagine three product pages each have a different primary button. The buttons use slightly different colors, spacing, and focus behavior.

An improvement could look like this:

1. **Understand the need:** Confirm what actions the buttons perform and which differences are intentional.
2. **Agree on the shared rule:** Decide what a primary action should look and behave like.
3. **Create tokens:** Name the shared action color, spacing, and corner shape by purpose.
4. **Build the component:** Give the button a small API, such as a label, a click action, and a disabled state.
5. **Check accessibility:** Ensure it is a real button, has a visible focus state, and works with a keyboard.
6. **Test and document:** Show its states and explain when to use it.
7. **Adopt it gradually:** Replace existing buttons where the shared behavior fits, and investigate cases that need something different.

The goal is not to force every action into one appearance. The goal is to make intentional choices easier and accidental differences less likely.

## 12. Study exercises

1. Pick a website you use. List three repeated interface patterns and where they appear.
2. Choose one repeated pattern. Write down its job, its inputs, and the actions it reports.
3. Find three repeated colors or spacing values in a project. Give each a name based on its purpose.
4. Sketch a folder structure for a small application. Explain why each folder exists.
5. Use a keyboard only to navigate a page. Write down any places where focus is hard to see or use.
6. Propose a change to a shared component. Identify who might be affected and what you would test.
7. Find a component with many options. Decide which options are essential and which reflect unrelated use cases.

## 13. Quick glossary

| Term | Simple meaning |
| --- | --- |
| Architecture | The way parts of an application are organized and connected |
| Component | A reusable piece of interface with a defined job |
| Design system | Shared design decisions, components, guidance, and practices |
| Design token | A named value for a design choice, such as a color or spacing size |
| Dependency | A relationship where one part of the code needs another part |
| API | The set of inputs and behaviors that a component exposes to its users |
| Accessibility | Designing and building so people with different abilities can use the product |
| Theme | A set of values that changes the visual style while keeping the same design roles |
| Regression | A problem introduced in something that used to work |

## 14. Project checklist

Before calling a frontend structure or shared component ready, ask:

- Is its purpose clear?
- Can a new team member find it?
- Does it depend only on code that makes sense for its role?
- Are its inputs and behavior easy to understand?
- Does it solve a real repeated need?
- Are design values shared where consistency matters?
- Does it work with a keyboard and assistive technology?
- Have important states and screen sizes been checked?
- Is there a useful example or explanation?
- Can the team update it without surprising other users?

## 15. Architecture patterns and trade-offs

There is no single folder structure or architecture that is best for every frontend. Choose a structure that fits the size of the product, the number of teams, and how often parts need to change independently.

### A small application

A small app can start with pages, a few shared components, and a small set of styles. Avoid creating packages, abstraction layers, or a design-system team before there is a real need. A simple structure is easier to understand while the product is still changing quickly.

### A modular frontend

As features grow, group code by the user capability it supports. A feature folder can keep its components, state, tests, and API interactions together:

```text
src/
  app/                 # Application setup, routes, and global providers
  features/
    catalog/
      components/
      data/
      catalog.test.ts
    account/
      components/
      data/
  shared/
    components/        # Reused across unrelated features
  foundations/
    styles/
    tokens/
```

This is an example, not a required template. Avoid folders with vague names such as `misc` that become a home for anything difficult to place. Keep a feature's internal details private where practical, and expose a small entry point for other features to use.

### Monolith, modular monolith, and micro-frontends

- A **frontend monolith** is one application built and released as a unit. It can be a good choice for one product and one team.
- A **modular monolith** is still one application, but it has clear internal feature boundaries. It is often a useful step as the codebase grows.
- **Micro-frontends** let separately owned frontend parts be developed or released independently and then composed into a product.

Independent deployment can help large organizations coordinate teams, but it adds runtime integration, shared-dependency, monitoring, accessibility, and user-experience complexity. Do not choose micro-frontends just because the codebase has many folders. First check whether clear modules and team ownership inside one application solve the problem.

### Separate design, application, and infrastructure concerns

Keep visual rules and reusable UI separate from feature-specific product decisions. Keep build and deployment configuration separate from page behavior. This makes it easier to tell whether a change belongs in a shared system, one feature, or the application's runtime setup.

## 16. Data, state, and service boundaries

An interface commonly handles several kinds of state. Treating them all as one kind of state often makes an application harder to reason about.

| State type | Example | Typical owner |
| --- | --- | --- |
| Local interaction state | Whether a tooltip is open | The component using it |
| Page or URL state | Selected tab, search query, page number | The route or URL |
| Server state | Product details fetched from an API | A data-fetching layer or cache |
| Cross-feature client state | Current locale or a temporary draft shared across routes | A deliberately chosen application-level store |

### Keep the source of truth clear

A source of truth is the place that owns the authoritative value. If the current search query is in the URL, avoid separately maintaining a second unsynchronized copy in several components. If data comes from a server, remember that a browser copy may be stale and needs loading, error, and refresh behavior.

### Separate rendering from network details

Components are easier to reuse when they receive the data and actions they need instead of constructing URLs, parsing every server response, and deciding application-wide error policy themselves. A feature-level data module can translate between API responses and the values the UI needs.

For example, a product screen might have these responsibilities:

```text
Route: reads product ID from the URL
Feature data module: loads product and reports loading/error states
Page component: arranges the product screen
Shared components: display headings, buttons, and status messages
```

Do not add a data layer solely for ceremony. Add one when it clarifies ownership, avoids duplication, or gives the application a consistent place to handle API concerns.

### Handle the full request lifecycle

Network requests can be loading, successful, empty, failed, or no longer relevant because the user navigated elsewhere. Design the interface for each meaningful state. Avoid showing stale results as if they belonged to a newer search. Provide a retry path when retrying is safe, and explain errors in language that helps the user recover.

## 17. Performance, resilience, and responsive behavior

Performance is part of the user experience. A page that eventually looks correct but takes too long to become useful is still a poor experience.

### Measure before optimizing

Use browser developer tools and real-user measurements to find slow routes, large assets, expensive rendering, or delayed interactions. Google's Core Web Vitals are a useful set of user-centered measurements: loading experience (LCP), responsiveness (INP), and visual stability (CLS). A score is a signal to investigate, not a substitute for understanding what users experience.

Common improvements include:

- Compressing and correctly sizing images.
- Loading only the JavaScript needed for the current route.
- Splitting large bundles at sensible route or feature boundaries.
- Avoiding unnecessary repeated renders or expensive calculations.
- Reserving space for images and other content to reduce layout shifts.
- Caching static assets with an appropriate update strategy.

More tools and optimizations are not automatically faster. Measure the result and check that the change did not make the application harder to maintain.

### Design for slow or unavailable services

Assume a network request can be delayed or fail. Provide clear loading feedback, useful error messages, retry behavior where appropriate, and a meaningful empty state. Keep actions safe when users click twice or revisit a page. Do not display a success message until the action has actually succeeded.

### Make layouts work at different sizes

Responsive design means the interface adapts to available space and input methods. Start with the content and task: decide what must remain visible, what can wrap or stack, and when navigation needs to change. Test with narrow viewports, zoom, long text, and keyboard navigation, not only a desktop screenshot.

Use flexible layout tools such as CSS Grid and Flexbox, and avoid assuming every user has a mouse or a large screen. Breakpoints should respond to where the layout stops working, not to a list of specific device models.

## 18. Design-system delivery and versioning

Shared components may be used by several applications or teams. Their release process should help adopters understand what changed and how to respond.

### Package boundaries

A design system can be a folder inside an application, a workspace package in a monorepo, or a published package. Choose based on who needs to reuse it and how independently it needs to be tested and released. A separate package introduces versioning and publishing work, so use one when those boundaries provide a real benefit.

Keep public exports intentional. If every internal file becomes a supported import path, later refactoring can break consumers. Provide documented imports and keep implementation details private.

### Compatibility and releases

Treat a change as potentially breaking if it removes or renames a public property, changes a component's meaning, or alters behavior that consumers rely on. Before a breaking change:

1. Identify consumers and the use cases affected.
2. Prefer an additive migration when it is practical.
3. Provide a clear deprecation period or migration instructions.
4. Test the new version in representative consuming applications.
5. Publish release notes that explain impact, not just file names.

Semantic Versioning is a common convention for communicating compatibility in published packages. It helps only when maintainers consistently define and follow what counts as a breaking change.

### Keep documentation close to the implementation

Examples should use the public API that consumers are expected to use. When a component changes, update its documentation and tests in the same change. A component catalog can make it easier to view variations without navigating the full product, but it does not replace documentation about real usage decisions.

## 19. Common problems and how to avoid them

| Problem | Why it hurts | Better approach |
| --- | --- | --- |
| Building a large library before teams need it | Adds maintenance work without proving value | Start with repeated, well-understood needs |
| A universal component with dozens of options | Makes valid behavior difficult to understand | Split by distinct purpose or use composition |
| Copying styles into every feature | Small differences accumulate and fixes must be repeated | Share stable tokens and patterns |
| Putting all state in a global store | Hides ownership and couples unrelated pages | Keep state near its users; promote only when needed |
| Importing feature code into low-level shared UI | Reverses dependency direction | Pass data and callbacks into shared components |
| Treating a visual catalog as complete documentation | Shows appearance but may not explain usage or accessibility | Document intent, behavior, and limitations |
| Rewriting all old screens at once | Makes the change large and risky | Migrate gradually and verify high-use flows |
| Measuring success only by component adoption | Reuse can be forced or inappropriate | Also measure usability, consistency, quality, and workarounds |
| Optimizing before measuring | Can add complexity without improving the user experience | Profile first and compare results after changes |

### A simple decision test for reuse

Before promoting a feature component to shared UI, ask:

1. Do multiple consumers need the same behavior, not just similar markup?
2. Are the important differences understood?
3. Can the component have a small, stable API?
4. Is there an owner who can maintain it and respond to issues?
5. Will sharing make changes safer or faster overall?

If the answers are uncertain, keep the component local until the repeated need becomes clearer.

## 20. Learning resources

These links point to standards, documentation, and practical tools. Standards and tool documentation can change, so check the current version when applying them to a project.

### Start with web fundamentals

- [MDN Learn Web Development](https://developer.mozilla.org/en-US/docs/Learn_web_development) — Tutorials and reference material for HTML, CSS, JavaScript, and the browser.
- [MDN: CSS building blocks](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Styling_basics) — A structured introduction to selectors, cascade, layout, and styling.
- [React: Learn](https://react.dev/learn) — Official React concepts and examples. If using another framework, use its official guide for the same topics: components, state, events, and composition.

### Architecture and reusable UI

- [Storybook documentation](https://storybook.js.org/docs) — Build, inspect, test, and document UI components in isolation.
- [GOV.UK Design System](https://design-system.service.gov.uk/) — A practical, well-documented example of shared components and patterns, including when and how to use them.
- [Material Design 3](https://m3.material.io/) — Design guidance and component patterns from Google's Material design system.
- [Frontend Architecture for Design Systems by Micah Godbolt](https://www.google.com/search?kgmid=%2Fg%2F11bwfv9hb1&q=Frontend+Architecture+for+Design+Systems) — The book named in the original request; use the publisher's edition for its full text. This guide is an original companion, not a substitute or reproduction.

### Design tokens and CSS

- [Design Tokens Community Group format](https://www.designtokens.org/tr/drafts/format/) — A draft format for exchanging design tokens between tools. It is a community-group draft, not a final W3C standard.
- [MDN: CSS custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Cascading_variables/Using_custom_properties) — Reference for CSS variables, often used to implement design tokens.
- [MDN: CSS Grid Layout](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Grid_layout) — Guide to two-dimensional page and component layouts.

### Accessibility and quality

- [Web Content Accessibility Guidelines (WCAG) 2.2](https://www.w3.org/TR/WCAG22/) — W3C accessibility standard. It is detailed; use it alongside practical testing and user feedback.
- [ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/) — Patterns and keyboard interaction guidance for common interface widgets. Prefer native HTML controls when they provide the needed behavior.
- [Testing Library: Guiding Principles](https://testing-library.com/docs/guiding-principles/) — Explains how to write tests around the way people use an interface.
- [Playwright documentation](https://playwright.dev/docs/intro) — Browser-based end-to-end testing across realistic user flows.

### Performance

- [web.dev: Learn Performance](https://web.dev/learn/performance/) — Lessons on measuring and improving web performance.
- [Web Vitals](https://web.dev/articles/vitals) — Explanation of Core Web Vitals and related user-experience measurements.

### Suggested order of study

1. Learn HTML, CSS, JavaScript, and browser basics.
2. Build a small application and organize it into a few clear features.
3. Identify repeated UI patterns and create a small set of shared components.
4. Introduce meaningful design tokens and document component usage.
5. Add accessibility, component, and end-to-end checks.
6. Measure real user experience and improve the most important problems first.

If several answers are no, simplify the design or improve the guidance before adding more options.