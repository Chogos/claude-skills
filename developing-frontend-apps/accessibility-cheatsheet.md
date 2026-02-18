# Accessibility Cheatsheet

## WCAG 2.2 Level AA Checklist

Organized by the four POUR principles:

| Principle | Guideline | Key requirements |
|-----------|-----------|-----------------|
| **Perceivable** | 1.1 Text alternatives | Alt text on images, labels on inputs |
| | 1.2 Time-based media | Captions for video, audio descriptions |
| | 1.3 Adaptable | Semantic structure, correct heading hierarchy, meaningful sequence |
| | 1.4 Distinguishable | Color contrast (4.5:1), text resizable to 200%, no info by color alone, reflow at 320px |
| **Operable** | 2.1 Keyboard | All functionality via keyboard, no keyboard traps |
| | 2.2 Enough time | Adjustable time limits, pause/stop/hide auto-updating content |
| | 2.4 Navigable | Skip links, descriptive page titles, focus order, visible focus, multiple ways to find pages |
| | 2.5 Input modalities | Target size ≥ 24x24px, dragging alternatives, accessible authentication |
| **Understandable** | 3.1 Readable | Page language set (`lang` attribute), abbreviation expansion |
| | 3.2 Predictable | Consistent navigation, consistent identification, no change on focus/input |
| | 3.3 Input assistance | Error identification, labels/instructions, error suggestion, error prevention for legal/financial |
| **Robust** | 4.1 Compatible | Valid HTML, name/role/value for custom widgets, status messages via ARIA live regions |

## Semantic HTML Elements

Use the right element — it provides keyboard support and screen reader semantics for free:

| Element | Implicit role | Use case |
|---------|---------------|----------|
| `<button>` | `button` | Triggers an action. Not `<div onClick>`. |
| `<a href>` | `link` | Navigates to a URL. Not for actions — use `<button>`. |
| `<input type="checkbox">` | `checkbox` | Boolean toggle. |
| `<input type="radio">` | `radio` | Selection from a group. |
| `<select>` | `combobox` / `listbox` | Dropdown selection. |
| `<nav>` | `navigation` | Site navigation links. |
| `<main>` | `main` | Primary page content. One per page. |
| `<aside>` | `complementary` | Sidebar, related content. |
| `<header>` | `banner` | Page or section header. (banner only when top-level) |
| `<footer>` | `contentinfo` | Page or section footer. (contentinfo only when top-level) |
| `<section>` | `region` (with label) | Thematic grouping with a heading. |
| `<form>` | `form` | Form container. |
| `<table>` | `table` | Tabular data. Not for layout. |
| `<h1>`-`<h6>` | `heading` | Document outline. Don't skip levels. |
| `<ul>`, `<ol>` | `list` | Screen readers announce item count. |
| `<dialog>` | `dialog` | Modal or non-modal dialog. Built-in focus trapping. |
| `<details>` | `group` | Expandable/collapsible content. No JS needed. |
| `<fieldset>` + `<legend>` | `group` | Groups related form controls with a label. |
| `<output>` | `status` | Result of a calculation or user action. |

## ARIA Roles Reference

### Landmark roles

Define page regions. Screen reader users jump between landmarks:

| Role | Equivalent HTML | Use case |
|------|----------------|----------|
| `banner` | `<header>` (top-level) | Site header with logo, search |
| `navigation` | `<nav>` | Navigation links |
| `main` | `<main>` | Primary content area |
| `complementary` | `<aside>` | Supporting content |
| `contentinfo` | `<footer>` (top-level) | Copyright, footer links |
| `search` | `<search>` | Search form |
| `form` | `<form>` (with label) | Named form area |
| `region` | `<section>` (with label) | Named generic region |

Label duplicate landmarks to distinguish them: `<nav aria-label="Primary">`, `<nav aria-label="Footer">`.

### Widget roles

For custom interactive components that have no HTML equivalent:

| Role | Use case | Required states |
|------|----------|----------------|
| `tablist` / `tab` / `tabpanel` | Tab interface | `aria-selected`, `aria-controls` |
| `menu` / `menuitem` | Action menus (not navigation) | `aria-expanded` on trigger |
| `tree` / `treeitem` | Hierarchical lists | `aria-expanded`, `aria-selected` |
| `listbox` / `option` | Custom select dropdown | `aria-selected`, `aria-activedescendant` |
| `slider` | Range input | `aria-valuemin`, `aria-valuemax`, `aria-valuenow` |
| `spinbutton` | Numeric stepper | `aria-valuemin`, `aria-valuemax`, `aria-valuenow` |
| `switch` | On/off toggle (different from checkbox) | `aria-checked` |
| `tooltip` | Descriptive popup on hover/focus | `aria-describedby` on trigger |
| `alertdialog` | Confirmation dialogs requiring action | Focus trap, `aria-describedby` |

### Live regions

Announce dynamic content changes to screen readers:

| Attribute | Behavior | Use case |
|-----------|----------|----------|
| `role="alert"` | Assertive — interrupts current speech | Error messages, urgent notifications |
| `role="status"` | Polite — announced after current speech | Success messages, progress updates |
| `aria-live="polite"` | Announced at next pause | Non-urgent updates (chat messages, counters) |
| `aria-live="assertive"` | Interrupts immediately | Urgent errors, time-sensitive info |
| `aria-atomic="true"` | Reads entire region on change | Counters, clocks (re-reads full content) |
| `aria-relevant="additions"` | Only announces added nodes | Chat logs, notification lists |

## ARIA States and Properties

| Attribute | Type | Use case |
|-----------|------|----------|
| `aria-expanded` | `true` / `false` | Collapsible sections, menus, dropdowns |
| `aria-selected` | `true` / `false` | Tabs, listbox options, grid cells |
| `aria-checked` | `true` / `false` / `mixed` | Checkboxes, switches, radio buttons |
| `aria-pressed` | `true` / `false` / `mixed` | Toggle buttons (bold, italic) |
| `aria-disabled` | `true` / `false` | Disabled but still visible/focusable (unlike `disabled` attribute) |
| `aria-hidden` | `true` / `false` | Hide from assistive tech (decorative icons, duplicated text) |
| `aria-invalid` | `true` / `false` | Form fields with validation errors |
| `aria-required` | `true` / `false` | Required form fields |
| `aria-describedby` | ID reference | Links input to error message, help text |
| `aria-labelledby` | ID reference(s) | Labels element using other visible text |
| `aria-label` | String | Labels element when no visible text exists |
| `aria-controls` | ID reference | Identifies the element this one controls |
| `aria-owns` | ID reference(s) | Associates elements not in DOM parent-child relationship |
| `aria-current` | `page` / `step` / `true` | Current item in navigation, breadcrumbs, steps |
| `aria-haspopup` | `true` / `menu` / `dialog` | Button opens a popup/menu/dialog |
| `aria-busy` | `true` / `false` | Region is being updated (defer announcements) |
| `aria-activedescendant` | ID reference | Focus management in composite widgets (listbox, tree) |

## Keyboard Interaction Patterns

| Component | Keys | Behavior |
|-----------|------|----------|
| **Button** | `Enter`, `Space` | Activate |
| **Link** | `Enter` | Navigate |
| **Checkbox** | `Space` | Toggle checked state |
| **Radio group** | `Arrow Up/Down` | Move selection between options |
| **Tabs** | `Arrow Left/Right` | Switch between tabs |
| | `Home` / `End` | First / last tab |
| **Menu** | `Arrow Up/Down` | Navigate items |
| | `Enter` / `Space` | Activate item |
| | `Escape` | Close menu, return focus to trigger |
| **Dialog** | `Tab` | Cycle focus within dialog (trapped) |
| | `Escape` | Close dialog, return focus to trigger |
| **Combobox** | `Arrow Down` | Open listbox / next option |
| | `Arrow Up` | Previous option |
| | `Enter` | Select option |
| | `Escape` | Close listbox |
| **Accordion** | `Enter` / `Space` | Expand/collapse section |
| | `Arrow Up/Down` | Move between headers |
| **Tree** | `Arrow Right` | Expand node / move to first child |
| | `Arrow Left` | Collapse node / move to parent |
| | `Arrow Up/Down` | Previous / next visible node |
| **Slider** | `Arrow Left/Down` | Decrease value |
| | `Arrow Right/Up` | Increase value |
| | `Home` / `End` | Min / max value |
| | `Page Up/Down` | Large increment/decrement |

## Color Contrast Requirements

| Context | Minimum ratio | Example |
|---------|---------------|---------|
| Normal text (<18px or <14px bold) | **4.5:1** | `#595959` on `#ffffff` |
| Large text (≥18px or ≥14px bold) | **3:1** | `#767676` on `#ffffff` |
| UI components and graphical objects | **3:1** | Borders, icons, focus indicators |
| Non-text contrast (Level AAA) | **4.5:1** | Enhanced for all UI elements |

Tools: Chrome DevTools contrast checker, `axe-core`, [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/).

Don't rely on color alone. Combine with icons, underlines, patterns, or text labels.

## Focus Management Rules

1. **Visible focus indicator**. Default browser outline or a custom one with ≥ 3:1 contrast and 2px+ width:
   ```css
   :focus-visible {
     outline: 2px solid var(--color-focus);
     outline-offset: 2px;
   }
   ```
2. **Logical tab order**. Follow DOM order. Avoid `tabindex` > 0 — it breaks natural flow.
3. **Trap focus in modals**. Tab cycles between modal elements only. Escape closes. Focus returns to trigger on close.
4. **Move focus on route change**. After SPA navigation, move focus to the new page heading or main content.
5. **Skip link**. First focusable element should be "Skip to main content":
   ```html
   <a href="#main" class="skip-link">Skip to main content</a>
   <!-- ... navigation ... -->
   <main id="main" tabindex="-1">...</main>
   ```
6. **`tabindex="-1"`** makes an element programmatically focusable (via JS) but not part of the tab order. Use on headings, containers, or elements receiving focus via `element.focus()`.
7. **`tabindex="0"`** adds a non-interactive element to the tab order. Rarely needed — prefer semantic elements that are naturally focusable.

## Common Mistakes

- **`<div onClick>`** — not focusable, not announced as interactive. Use `<button>`.
- **Missing `alt` on `<img>`** — screen reader reads the filename. Add `alt=""` for decorative images.
- **Heading hierarchy skips** — jumping from `<h1>` to `<h3>`. Screen reader users navigate by heading level.
- **`aria-label` overused** — if visible text exists, use `aria-labelledby` to point at it. Duplication drifts.
- **`role="button"` on `<div>`** — requires manual `tabindex="0"`, `Enter`/`Space` handling, and `:focus` styles. Just use `<button>`.
- **Color as only indicator** — red text for errors without an icon or "Error:" prefix. Colorblind users miss it.
- **Auto-playing media** — disruptive to screen readers and users with cognitive disabilities. Provide pause control.
- **Missing `lang` attribute** — `<html lang="en">`. Screen readers use this to select the correct pronunciation engine.
- **Form errors with no association** — error text near the input but not linked with `aria-describedby`. Screen reader users don't know the error exists.
- **Custom dropdown with no keyboard support** — mouse-only components exclude keyboard and switch users.
- **Disabled buttons with no explanation** — use `aria-describedby` to explain why a button is disabled, or keep it enabled and show validation on click.
- **Infinite scroll with no alternative** — keyboard users and screen readers can't reach content below the fold. Provide pagination or "Load more".