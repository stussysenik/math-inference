## ADDED Requirements

### Requirement: LiveSvelte component suite

The system SHALL ship a LiveSvelte component suite in slice 1 containing at least `KatexBlock.svelte`, `VerificationPill.svelte`, `SectionStream.svelte`, `PromptComposer.svelte`, and `HypertimeScrubber.svelte`. Each component SHALL receive state via LiveSvelte props from the server and SHALL NOT hold authoritative state locally.

#### Scenario: KatexBlock renders LaTeX source

- **WHEN** `KatexBlock.svelte` is mounted with the prop `latex: "\\int_0^1 x^2 dx"`
- **THEN** the component renders the expression with KaTeX in display mode and exposes a copy-to-clipboard button for the raw LaTeX source

#### Scenario: SectionStream receives server updates

- **WHEN** LiveView pushes a new section into the `sections` prop
- **THEN** `SectionStream.svelte` appends the new section to its rendered list with an entrance animation, without clearing or re-animating existing sections

#### Scenario: PromptComposer submits on Cmd+Enter

- **WHEN** the user focuses the composer textarea and presses Cmd+Enter (macOS) or Ctrl+Enter (Linux/Windows)
- **THEN** the component emits a `submit` event to the parent LiveView with the current prompt text and clears the textarea

### Requirement: GSAP motion system

The system SHALL use GSAP (GreenSock) as the sole motion library in the frontend. Motion durations SHALL be tuned between 180 and 220 milliseconds for micro-interactions and SHALL NOT exceed 400 ms for any animation. GSAP ScrollTrigger and Flip SHALL be available for the Hypertime scrubber and section reordering respectively.

#### Scenario: Section entrance animation

- **WHEN** a new section is appended to `SectionStream.svelte`
- **THEN** the component SHALL animate the section with a 200 ms GSAP tween that fades opacity from 0 to 1 and translates Y from 8 px to 0

#### Scenario: Verification pill state transition

- **WHEN** `VerificationPill.svelte` receives a new `status` prop value
- **THEN** the component SHALL animate the pill's background and icon with a 180 ms GSAP tween rather than snapping instantly

#### Scenario: Reduced motion is respected

- **WHEN** the user's OS reports `prefers-reduced-motion: reduce`
- **THEN** the system SHALL set GSAP's global duration to 0 via `gsap.globalTimeline.timeScale()` or an equivalent mechanism, so animations become instant state changes

### Requirement: Phosphor icons via unplugin-icons

The system SHALL use `unplugin-icons` configured with the `@iconify-json/ph` package for all iconography. Icons SHALL be imported via the `~icons/ph/<name>` convention so that Vite can tree-shake unused icons at build time.

#### Scenario: Icon is imported and rendered

- **WHEN** a Svelte component imports `import CheckCircle from '~icons/ph/check-circle-duotone'` and renders `<CheckCircle class="w-5 h-5 text-emerald-400" />`
- **THEN** the build output includes the inline SVG for that one icon and no other Phosphor icons

#### Scenario: Six weight variants are available

- **WHEN** a developer needs a bold variant of an icon
- **THEN** importing `~icons/ph/arrow-right-bold` SHALL work without additional configuration, and similarly for `-thin`, `-light`, `-regular`, `-fill`, and `-duotone` suffixes

### Requirement: Tailwind v4 with SaladUI primitives

The system SHALL use Tailwind CSS v4 with the `@theme` directive for design tokens. The system SHALL use SaladUI (a shadcn/ui port for LiveView) as the base primitive library for dialogs, tooltips, command palettes, tabs, and other a11y-critical components.

#### Scenario: Tailwind v4 is configured via app.css

- **WHEN** a developer inspects `assets/css/app.css`
- **THEN** the file SHALL begin with `@import "tailwindcss";` and SHALL declare a `@theme` block defining the Railway-aesthetic design tokens (color palette, font stack, border radii, animation durations)

#### Scenario: SaladUI primitives available to components

- **WHEN** a LiveView template needs a tooltip or dialog
- **THEN** the developer SHALL use the corresponding SaladUI function component rather than writing raw HTML + ARIA attributes

### Requirement: Railway-aesthetic design token layer

The system SHALL ship a custom design token layer in `assets/css/app.css` that evokes the Railway developer platform: deep obsidian backgrounds, 1 px gradient borders, monospace uppercase tracking-wide labels, status pills with subtle glow, terminal-style blocks for tool call traces, and dark-first color palette.

#### Scenario: Dark mode is the default

- **WHEN** a fresh user loads the site with no saved preference
- **THEN** the root element SHALL render in dark mode and SHALL NOT flash white before a theme script runs

#### Scenario: Status pills carry engine-specific colors

- **WHEN** a `VerificationPill` is rendered for the `:verified` state
- **THEN** the pill SHALL use a green accent with a 10% glow halo, matching the Railway "success" token in `app.css`

#### Scenario: Terminal-style traces use monospace

- **WHEN** the `ToolCallTimeline` component renders a tool call
- **THEN** the trace lines SHALL use the Geist Mono typeface and a fixed dark background with 1 px gradient border

### Requirement: Outline interaction design language

The system SHALL favor outlined interactive affordances over filled ones. Interactive elements SHALL use 1 px strokes with gradient borders, ghost buttons that reveal an outline on hover, dashed borders for pending or loading states, and outlined Phosphor icons rather than filled variants.

#### Scenario: Ghost button hover state

- **WHEN** a user hovers over a ghost button
- **THEN** the button SHALL reveal a 1 px outline transitioning in over 180 ms and SHALL NOT flip to a filled background

#### Scenario: Pending state uses dashed border

- **WHEN** a section is in the `:verifying` status and its skeleton placeholder is visible
- **THEN** the placeholder SHALL use a dashed 1 px border to signal transient state

### Requirement: Focus-visible ring system

The system SHALL render a 2 px outlined focus ring with a 2 px offset on every interactive element when the element is focused via keyboard navigation. The ring SHALL use an accent color at 60 % opacity with a soft glow and SHALL animate in over 150 ms. The system SHALL use `:focus-visible` to distinguish keyboard focus from mouse clicks.

#### Scenario: Keyboard focus shows the ring

- **WHEN** a user presses Tab to focus a button
- **THEN** the button SHALL display the 2 px outlined ring with accent color and glow

#### Scenario: Mouse click does not show the ring

- **WHEN** a user clicks a button with the mouse
- **THEN** the button SHALL NOT show the keyboard focus ring because `:focus-visible` does not match for mouse interactions

#### Scenario: Every interactive element has a focus state

- **WHEN** a reviewer audits the component suite
- **THEN** every button, link, input, select, textarea, and custom interactive widget SHALL have a defined `:focus-visible` style — no element SHALL rely on the browser default

### Requirement: Hypertime scrubber UI

The system SHALL implement a Hypertime scrubber component (`HypertimeScrubber.svelte`) that lets the user drag a timeline to replay any moment in the current or past generation. Scrubbing SHALL use GSAP ScrollTrigger or an equivalent pointer-drag mechanism, and scrubbing backward SHALL animate section state transitions in reverse.

#### Scenario: Scrubbing backward replays sections in reverse

- **WHEN** the user drags the scrubber backward past three section-complete events
- **THEN** the three sections SHALL visually regress their status pills from `:verified` back to `:verifying` and then to `:pending`, animated in reverse order

#### Scenario: Scrubbing forward replays sections in original order

- **WHEN** the user drags the scrubber forward after previously scrubbing back
- **THEN** the sections SHALL re-emerge in their original order with their original entrance animations

#### Scenario: Hypertime state is visible in the UI

- **WHEN** the user has scrubbed to a past moment and is no longer at the current time
- **THEN** the UI SHALL show a "replaying" indicator with a button to return to live, and SHALL disable the prompt composer until the user returns to live

### Requirement: Keyboard-first command palette

The system SHALL provide a command palette triggered by Cmd+K (macOS) or Ctrl+K (Linux/Windows) that allows the user to jump between sections, switch models, export a generation, toggle Hypertime replay, and invoke other session-level actions without touching the mouse.

#### Scenario: Command palette opens on Cmd+K

- **WHEN** the user presses Cmd+K anywhere in the app
- **THEN** the command palette SHALL open as a centered modal with the search input focused

#### Scenario: Command palette filters as the user types

- **WHEN** the user types into the command palette search input
- **THEN** the command list SHALL filter to matches in real time with fuzzy matching, and the first match SHALL be visually selected by default

#### Scenario: Command palette closes on Escape

- **WHEN** the user presses Escape while the palette is open
- **THEN** the palette SHALL close and focus SHALL return to the element that was focused before the palette opened
