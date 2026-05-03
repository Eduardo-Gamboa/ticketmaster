# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file, client-only prototype that simulates the Ticketmaster MX experience for the BTS "ARIRANG" 2026-27 world tour. UI is in Spanish. There is no build system, bundler, package manager, or test suite — everything is HTML, CSS, and vanilla JavaScript inlined into `index.html` (~6.4k lines).

## Running

Open `index.html` directly in a browser, or serve the directory with any static file server (e.g. `python -m http.server`, `npx serve`). There are no dependencies to install and no build step.

Reference assets sit next to `index.html`:
- `*.mhtml` — saved pages from the real Ticketmaster MX site used as visual/behavior reference.
- `*.jpg` / `*.jpeg` / `*.png` — UI mockups and screenshots the user pastes in to request changes.

## Architecture

Everything lives inside one file. The layout of `index.html` is:

1. **`<style>` block (lines ~8–4170).** Organized into labeled sections via `/* ========== SECTION ========== */` banners (RESET & BASE, TYPOGRAPHY SCALE, TOP UTILITY BAR, MAIN HEADER / NAV, HERO BANNER, EVENT LISTINGS, EVENT DETAIL PAGE, ACCOUNT / MY TICKETS PAGE, TICKET DETAIL PAGE, TICKET VISUAL CONSOLIDATED, TRANSFER SUCCESS BANNER, EVENT INFO PANEL, TRANSFER FORM PAGE, NEW TRANSFER FORM — TOPBAR & CARDS, MAP MODAL, PROFILE PICKER, TRANSFER SUCCESS SCREEN, BARCODE MODAL, TRANSFER ERROR PAGE, etc.). When editing styles, find the matching banner first — class names are scoped by convention (e.g. `.tfp-*` = transfer form page, `.tdp-*` = ticket detail page, `.tsm-*` = transfer success modal, `.pp-*` = profile picker, `.bm-*` = barcode modal). Line numbers drift quickly — grep for the banner text rather than trusting offsets.
2. **Static DOM shell (lines ~4171–4345).** Top bar, main header, mobile nav overlay, search overlay, `<main id="main-content">` (index.html:4266), footer (index.html:4269). The shell is rendered once; views are injected into `#main-content`.
3. **Data constants inside `<script>` (opens at index.html:4346).** `events` (index.html:4348), `userTickets` (index.html:4577), and an `icons` object of inline-SVG strings (index.html:4587). The per-profile `profiles` map (index.html:5027) and the working `eventTickets` array (index.html:5051) live further down near the profile logic.
4. **Router + view renderers (lines ~4627–6110).** Hash-based SPA; `</script>` closes at index.html:6111.
5. **Stadium map modal — static HTML after `</script>` (lines ~6114–6422).** The `<div class="map-modal" id="mapModal">` (index.html:6114) and its inline SVG of Foro GNP are hardcoded into the page, not injected by a render function. To edit the stadium diagram or the highlighted section, edit the bottom of `index.html` directly (see commit `bbdbc6a`).

### Routing

Hash-based router at `router()` (index.html:4657), wired to `DOMContentLoaded` and `hashchange`. The router first gates on the profile picker: if `_currentProfile === null`, it renders the picker and returns before consulting the hash. Once a profile is selected, routes are:

- `#/` → `renderHome()`
- `#/event/:id` → `renderEventDetail(id)`
- `#/ticket/:id` → `renderTicketDetail(id)`
- `#/account` → `renderAccount()`
- `#/transfer` → `renderTransfer()`
- `#/transfer-error` → `renderTransferError()`

Each render function sets `document.title` and replaces `mainEl.innerHTML` with a template-literal-built view. Interactivity is wired through inline `onclick="..."` handlers calling top-level functions (e.g. `tfpToggle`, `submitTransfer`, `openBarcodeModal`, `selectProfile`). There is no framework, no component system, no virtual DOM.

### Profile picker

On first load (and on every reload, since state is in-memory only) the router shows `renderProfilePicker()` (index.html:4627) with two demo accounts defined in the `profiles` map (index.html:5027): **Laura Ortiz** (seats 16/17 in row 6) and **Francisca Manzaneres** (seats 5/8/9 in row 5), both in section NA27C. Seat lists in the `profiles` map change frequently — treat the file as the source of truth, not this doc. `selectProfile(id)` sets `_currentProfile`, copies `profiles[id].tickets` into the working `eventTickets` array, clears `_transferState`, resets `barcodeCurrentIndex`, and navigates to `#/account`. Helpers `getProfile()`, `profileName()`, `profileOrder()`, and `profileSeatsLabel()` read current profile state for views that need to display the user's name, order number, or seat list. There is intentionally no in-app profile switcher — returning to the picker requires a page reload (see commit `389de84`).

### Transfer flow state

Transfer state is deliberately stored in a module-level variable, **not** in `sessionStorage` or `localStorage`:

```js
let _transferState = null; // reset on every page reload
```

`submitTransfer()` mutates `_transferState` with `{ indices, email, name }`; `renderTicketDetail()` and `renderTransfer()` both read it to filter out already-transferred tickets and render the success banner. Reloading the page intentionally resets the demo. Preserve this behavior — a prior commit (`4095037`) explicitly moved away from `sessionStorage` for this reason.

When tickets are transferred in multiple passes, `submitTransfer()` **accumulates** indices into `_transferState.indices` rather than overwriting (see commit `75eb49a`). The transfer form must filter `eventTickets` against `_transferState.indices` so already-transferred seats don't reappear.

### Barcode modal

`openBarcodeModal` / `drawBarcode` / `updateBarcodeDisplay` implement a fake rotating barcode on a `<canvas>` with a 120-second countdown (`barcodeSecondsLeft`, `barcodeTimerInterval`). `changeBarcodeTicket(dir)` cycles through `eventTickets`. `stopBarcodeTimer()` must be called whenever the modal closes to avoid leaking intervals.

## Editing conventions

- **Spanish copy.** All user-facing strings are Spanish. Non-ASCII characters are written as `\u00XX` escapes inside JS template literals (e.g. `M\u00e9xico`, `s\u00e1b`) and as HTML entities inside HTML fragments (e.g. `&aacute;`, `&uacute;`). Match the style of the surrounding code.
- **Inline SVG icons.** Reuse entries from the `icons` object (index.html:4587) when possible; otherwise inline an SVG string literal the same way neighboring code does. Do not add an icon library.
- **CSS class prefixes.** New styles for an existing view go under that view's banner and should reuse its prefix. New views: add a new `/* ========== NAME ========== */` banner.
- **No dependencies.** Do not introduce npm, a build step, a framework, or external CDN scripts without being asked.
- **Do not apply the global Turistore `.NET` rules** (300-line file limits, MediatR Command/Handler/Validator, MudBlazor icons, etc.) to this project. Those rules are for a separate enterprise .NET/Blazor codebase and do not fit a single-file static HTML prototype. The "no emojis in UI/code" and "no hardcoded credentials" rules still apply.
