# PDF Widget — Design

Date: 2026-08-15

## Goal

An Android home-screen widget that renders a page of a PDF and serves as the
entire reading surface. The user unlocks the phone and the page they left off on
is already on screen. Reading happens on the home screen; the app has no reader
UI of its own.

## Non-goals

These were considered and explicitly excluded from v1:

- A full-screen viewer activity. Tapping the page does not open anything.
- A document library, folder watching, or thumbnail browser.
- Jump-to-a-specific-page input.
- Touch gestures on the page. Widgets receive clicks only; all navigation is by
  button.
- Zotero synchronisation. Deferred, but the state layer is shaped so a remote
  position source can be added without restructuring.
- EPUB and image formats.

## User experience

The widget is resizable and expected to be placed large — most of a home screen
panel. It has two visual states.

**Collapsed** (the reading default): the rendered page, the current page number
in a corner, and a single control-toggle button.

**Expanded**: the same, plus a control cluster overlaying the page:

- Previous page / next page
- Zoom in / zoom out
- Four-way pan pad
- Night mode toggle
- Switch document

Controls are flat, translucent, and low-contrast so they recede into the page.
No gradients, bevels, or drop shadows.

Selecting a document opens the system file picker over the home screen. On
return, the widget shows the chosen PDF. This is the only flow that leaves the
home screen.

## Architecture

| Module | Responsibility |
|---|---|
| `Viewport` | Pure Kotlin. Zoom level, normalised offset, clamping, pan stepping, page rollover. No Android imports. Carries the test suite. |
| `PdfPageRenderer` | Wraps `PdfRenderer`. Given document, page index, viewport and target size, returns a bitmap. |
| `WidgetStateStore` | DataStore persistence for per-widget state and per-document reading positions. |
| `WidgetAction` | Enum of every button action. |
| `PdfWidget` | Glance composition. Reads state, invokes the renderer, lays out page and controls, dispatches actions. |
| `DocumentPickerActivity` | Transparent, no-UI activity that runs `ACTION_OPEN_DOCUMENT`, takes persistable permission, writes state, updates the widget, finishes. |
| `RenderScheduler` | Serialises all render work onto a single dispatcher behind a mutex, and holds the cached open document. |

Every tunable value (zoom stops, pan step fraction, render timeout, cache size)
lives in a single constants object. No literals at call sites.

## State model

Two separate keyspaces, deliberately:

```
WidgetState        keyed by appWidgetId
  documentUri: Uri?
  controlsExpanded: Boolean
  colorMode: ColorMode        // SYSTEM | LIGHT | NIGHT

DocumentPosition   keyed by document URI
  pageIndex: Int
  zoomStopIndex: Int
  offsetX: Float              // normalised 0..1 of page width
  offsetY: Float              // normalised 0..1 of page height
```

Keying position by document URI rather than by widget means switching away from
a PDF and back restores where you were. Keying the rest by widget ID means
multiple widgets are independent from the first commit.

Offsets are stored normalised rather than in pixels so that resizing the widget
does not scramble the reading position.

## Viewport semantics

Zoom is a discrete ladder of multipliers applied to the fit-to-widget scale,
starting at fit (1.0). Zooming preserves the centre point of the current view.

Panning moves by 0.85 of the viewport dimension, so successive steps overlap by
15% and no line of text falls between two views.

Clamping and rollover:

- Horizontal panning clamps at the page edges. It never changes page.
- Panning down while already at the bottom edge advances to the next page with
  the vertical offset reset to the top; horizontal offset is preserved.
- Panning up while already at the top edge moves to the previous page with the
  vertical offset set to the bottom.
- At fit-to-widget zoom the page has no vertical slack, so panning down is
  immediately a page turn. This is intended and makes the down button mean
  "more text" at every zoom level.
- The page-turn buttons reset the vertical offset to the top and preserve zoom
  and horizontal offset.
- Rollover past the last page and before the first page is a no-op.

All of this is pure functions over immutable values and is where test effort
concentrates.

## Rendering pipeline

A click produces an action, which loads state, applies the pure viewport
transformation, persists the result, renders, and pushes new `RemoteViews` to
the launcher.

The target bitmap matches the widget's current pixel dimensions, derived from
the app widget options. Rather than rendering a full page at high scale and
cropping it, the visible region is rendered directly by passing a scale and
translate `Matrix` to `PdfRenderer.Page.render`, so cost stays proportional to
the widget, not to the zoom level.

Night mode applies an inverting colour matrix at draw time.

## Technical constraints and mitigations

- **RemoteViews bitmap memory.** Bitmaps cross a Binder transaction and the
  widget host caps total bitmap memory. Rendering at exactly widget size keeps
  this comfortable. If a launcher rejects it, fall back to writing PNGs to cache
  and using `setImageViewUri` with a `FileProvider`.
- **Tap latency.** Opening and rendering costs roughly 50–200 ms. `RenderScheduler`
  keeps the `ParcelFileDescriptor` and `PdfRenderer` open between actions, and
  speculatively pre-renders the next page.
- **`PdfRenderer` concurrency.** It is not thread-safe and permits one open page
  at a time. All access is serialised through `RenderScheduler`.
- **Non-seekable URIs.** Cloud-backed providers may not give a seekable
  descriptor. Copy to cache on first open and render from the copy.
- **Permission durability.** `takePersistableUriPermission` at pick time, so
  widgets survive reboot. If permission is later revoked, the widget renders an
  explicit "document unavailable, tap to reselect" state rather than failing
  silently.
- **Glance vs. RemoteViews.** Glance is the default for code clarity. If its
  bitmap handling proves unreliable at these sizes, drop to raw `RemoteViews`.
  The module boundaries are unchanged either way; only `PdfWidget` is affected.
- **Widget update work must not run on the main thread.** Actions complete
  asynchronously and the widget shows its previous frame until the new one is
  ready.

## Testing strategy

- `Viewport` is tested exhaustively with plain JVM unit tests, written first:
  zoom clamping at both ends of the ladder, pan clamping on all four edges,
  rollover forwards and backwards, no-op at document boundaries, offset
  preservation across page turns and resizes.
- `WidgetStateStore` is covered with Robolectric for round-tripping and for
  independence between widget IDs.
- `PdfPageRenderer` is covered against a small fixture PDF, asserting output
  dimensions and that a zoomed render differs from a fit render.
- Widget rendering inside a launcher is verified manually. There is no honest
  automated assertion available for it.

## Deferred

Zotero integration. Zotero syncs reading position as a synced setting keyed
`lastPageIndex_<library>_<attachmentKey>`, readable from the settings endpoint
of the Web API. Adding it later means introducing a position source behind the
existing `DocumentPosition` keyspace and a background refresh; no change to the
viewport, rendering, or widget layers.
