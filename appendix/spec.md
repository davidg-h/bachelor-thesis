# NELS Graph Panel

## Architecture context

- ComfyUI's frontend extension mechanism is JavaScript/TypeScript-based
  - Register the panel through ComfyUI's existing extension API (`app.registerExtension`)
  - Do not modify ComfyUI core
- DeepWiki analysis of the ComfyUI codebase (`Comfy-Org/ComfyUI`):
  - Custom nodes are discovered by scanning the `custom_nodes` directory for a `NODE_CLASS_MAPPINGS` export
  - A package can additionally export `WEB_DIRECTORY` to have its own JS/CSS served as static assets and auto-discovered by the frontend
  - Use this mechanism rather than a separate build/deploy path
- Official ComfyUI docs on sidebar tabs:
  - has dedicated `app.extensionManager.registerSidebarTab({ id, icon, title, type: "custom", render })` API for adding a persistent, dedicated panel to the ComfyUI
  - `render(element)` receives a plain DOM element to mount into
  - Use this (not a custom node widget) as the actual mounting point for the graph panel
  - Mount our own component tree into the provided element
- ComfyUI's own `PromptServer`:
  - Already runs an aiohttp server and exposes a WebSocket channel used for node-status and progress events
  - Reachable from the frontend via `api.addEventListener(eventName, handler)`
  - Reuse this existing channel with our own custom event name (e.g. `nels.session_event`) for pushing session/session-result updates
  - Do not open a separate WebSocket connection
- DeepWiki analysis of GitNexus (`abhigyanpatwari/GitNexus`):
  - Graph-based exploration UI over a large structured dataset (a knowledge graph of code relationships)
  - Rendered with Sigma.js + Graphology
  - User browses by selecting/hovering/filtering nodes in a spatial layout rather than scrolling a list or table
  - Adopt this general interaction paradigm, and Sigma.js + Graphology as the 2D rendering library, for exploring prompt embeddings
- Each dataset already stores both a 2D and a 3D UMAP projection of the same points
  - Offer both a 2D and a 3D view of the loaded graph rather than committing to only one
  - Let the user pick whichever is easier to read for a given dataset or task

## Tech stack

- Frontend language: TypeScript, built with Vite into a single-file bundle
  - The CSS also needs to be injected programmatically at runtime rather than shipped as a separate stylesheet, since ComfyUI will not auto-load one for a dynamically rendered tab
- Component/state framework: Preact with `@preact/signals`
  - No Redux/store framework
  - State should live in a handful of plain modules, each owning one concern, exporting signals and setter functions directly
- 2D graph rendering: Sigma.js, on top of graphology for the underlying graph data structure
- 3D graph rendering: Three.js
- The interaction vocabulary must be identical across the 2D and 3D renderer wherever practical, even though the underlying rendering technology differs
- 2D and 3D should never both be mounted at the same time
  - Switching view modes should fully tear down one renderer and build the other

## State architecture

Split state into focused modules rather than one global store:

- **Graph/dataset state**: the currently loaded set of points and their display coordinates, independent of which operation is active
  - Support both a 2D and a 3D coordinate projection of the same underlying data
  - A single view-mode switch re-derives the active display coordinates
  - Cache the resolved display coordinates per dataset configuration
  - Reuse the cached coordinates instead of trusting a freshly arrived snapshot when the configuration hasn't actually changed
  - Goal: routine reloads don't needlessly reposition every point (and don't require an expensive recomputation) unless the user explicitly asks for a fresh layout or the underlying configuration changed
  - Raw dataset coordinates are not display coordinates; map them into a shared, fixed display range before rendering
    - Derive the mapping from the bounding range of the currently loaded points, using one uniform scale factor across all axes rather than normalizing each axis independently
    - Goal: preserve the relative proportions of the projection (a cluster twice as wide as it is tall stays that shape) instead of stretching every axis to fill the same range
    - This mapping must be invertible, since operations in the backend requires raw coords rather than display position -> convert into raw coordinate space before sending it to the backend
- **Operation state**: the current operation type, its configuration/settings (alpha schedules, neighbor counts, etc.), and the last computed result from the backend
  - Treat this as its own concern, separate from the raw point data
- **Selection/hover state**: which point ids are currently selected, and which one (if any) is hovered
  - Keep independent of the operation and dataset state, so that selecting points doesn't require touching operation config, and vice versa
- **Session/sync state**: discovering and loading sessions from the backend, applying incoming state, and pushing local UI changes back to the backend
  - This is the orchestration layer
  - It should be the only place that talks to the API client and to the live event channel
- **Event/status log state**: a bounded log of backend events and a single current status string
  - Keep separate from everything else so that logging never becomes a bottleneck for the rest of the UI
- **Misc**: Avoid direct imports between the renderer modules (2D/3D) and the session/orchestration module in either direction
  - If the orchestration layer needs to reach "whichever renderer is currently active," introduce a small indirection layer for that
  - Do not import renderer code directly into session logic, or session logic into renderer code

## Backend communication

- All request/response calls (loading a session, pushing UI state, requesting a fresh preview, ...) go through a single thin REST client
  - The client proxies through ComfyUI's own fetch helper, not a separate HTTP client library
- All live push updates (new results, progress during long-running operations, newly available data) arrive through the shared WebSocket event channel described above
  - Dispatch by event type in one central place
- Define the full data contract (points, sessions, UI state, operation results) as shared TypeScript types
  - These types should mirror the backend's own data model field-for-field, so the two sides can be kept in sync by inspection rather than by convention
- Panel interactions (changing the selection, editing operation config, ...) never trigger a computation directly
  - Each interaction only pushes an incremental state update to the backend
  - The actual operation runs only when the user queues the ComfyUI graph itself, against a frozen snapshot of the interface state taken at that moment
  - Goal: further UI interaction while an operation is running never changes the inputs of that already-queued run

## Panel structure

- Register a single dedicated, resizable side panel (not a node-body widget) inside ComfyUI's own UI
- Provide a toggle between a graph view and a logs view
- Layout:
  - A main graph area (the 2D/3D canvas plus overlays)
  - A sidebar containing the operation configuration, the current result, and current session info
  - Each sidebar section lives in its own collapsible card, so the user can hide sections they aren't currently using
  - The sidebar is resizable by dragging its edge
  - The sidebar remembers its width across sessions (e.g. via local storage)
- The graph area needs supporting overlays beyond the raw point cloud:
  - A point-count indicator
  - A cluster legend and cluster hulls
  - An optional legend display placeholder; fillable with custom content by the active operation
  - A hover tooltip
  - A drag-select rectangle overlay
  - All of these are layered on top of the render surface without interfering with it
- Provide a way to switch between multiple known sessions/datasets
  - dropdown selector in the header
  - Each session/dataset is a self-contained set of points and clusters
  - Switching the active session via the dropdown alone is enough to load whatever is already known for that session; it does not require pressing reload/refresh
- Provide a separate reload/refresh control that re-syncs the list of known sessions and forces a fresh dataset preview for the currently active session, bypassing the coordinate cache described above so the layout actually updates
  - This is for when the user changed the dataset loader's configuration (path, sample count, seed, ...) and wants to see the update without queuing the full downstream pipeline
  - Outside of this explicit action, if a new snapshot arrives for the same session (e.g. from a live update) and the dataset configuration hasn't actually changed, keep using the cached layout instead of recomputing it - see the caching behavior described in State architecture above
- Provide a way to export the current view as a single, self-contained, shareable file
  - Include an image of the current view
  - Include a summary of the loaded dataset, clusters, and current operation result

## Interaction model

Must behave the same in both the 2D and 3D view wherever the underlying input technology allows it:

- Click selects a point, replacing the current selection
- Shift-, ctrl-, or meta-click adds or removes a point from the current selection instead of replacing it
- Shift-drag on empty background opens a rectangular marquee selection
  - Releasing it selects everything inside the rectangle
  - Treat a very small drag as a plain click rather than a selection, so accidental tiny drags don't clear an existing selection unintentionally
- Hovering a point shows a tooltip with that point's data (id, text, cluster, coordinates)
- Right-clicking a point opens a small context menu (at minimum, a way to copy the point's prompt text)
- Clicking empty background clears the current selection
  - Exception: if the active operation is in a special placement mode (see Latent Probe below), the click instead places a marker at that position
- In the 3D view specifically, dragging the camera (orbit/pan/zoom) must not conflict with shift-drag selection
  - Disable camera controls for the duration of a shift-drag and re-enable them afterward

## Panel-level controls and interactions

Beyond the graph canvas itself, the surrounding panel chrome needs its own set of controls:

- **Header**:
  - A session selector (switch which loaded dataset/session is active)
  - An adjacent reload/refresh control
  - A 2D/3D view toggle
  - An animate on/off toggle
  - A status indicator showing whether the LLM used for labeling/annotation is currently loaded, with a way to unload it on demand to free resources
  - An export action
- **Operation card**:
  - A dropdown to choose which navigation operation is currently active, which then reveals only that operation's own controls
  - A "clear" action that resets the current selection, result, and any in-progress operation-specific state together
  - A "fit view" action that re-frames the camera/viewport to the currently loaded points
- **Cluster legend**:
  - Collapsible open/closed
  - A toggle for showing/hiding cluster outline hulls
  - Clicking a cluster entry selects every point in that cluster and filters/dims the rest
  - Clicking the same entry again clears that filter
- **Result card**, once an operation has produced a result:
  - Clicking any individual listed result item (a neighbor hit, a ranked axis-projection entry, a generated variant, a role-labeled source point) selects the corresponding point on the canvas, always replacing the current selection rather than adding to it
  - For trajectory-producing operations, provide a per-step table where clicking a step highlights that specific generated point on the canvas
  - For trajectory-producing operations, provide a toggle for whether generated points should snap onto their real anchor points
  - For trajectory-producing operations, provide a toggle for showing or hiding the generated points/overlay on the canvas at all
  - For semantic axis projection specifically, provide a toggle that visually highlights just the projection path/trajectory, dimming everything else in the graph
- All sidebar sections (operation configuration, result, session info) should be individually collapsible, so the user can hide the ones they aren't currently using

## Operation results in the graph

- Do not render operation outputs (trajectory steps, generated variants, nearest-neighbor hits) in a separate results view
- Render them as additional points inside the same graph instead
  - Visually distinguish them from real dataset points (different texture/marker, a highlighted ring or outline, and a subtle idle animation) so they read as "generated" rather than "loaded"
  - Keep them in the same coordinate space as real points
  - Make them selectable/hoverable/right-clickable through the same interaction code path as real points, not a separate one
- Where a result implies an ordered path between points (an interpolation, a directed walk), draw a connecting line or curve between the relevant points/results
- Where a result is a set of independent items with no implied order (e.g. generated variants for a consistency check), do not draw a connecting path between them
- Where a result is an unordered set of per-result matches to a query point (e.g. nearest-neighbor hits), draw a dashed connection from each result back to its query point instead
  - Use the dashed style specifically to distinguish this from the solid ordered-path case above
- Provide a way to snap a generated point visually onto its real "anchor" point at the extremes of a schedule (e.g. the very start/end of an interpolation)
  - Goal: it's clear which real point a generated sequence begins or ends at, even while idle animation is active

## Interpretability affordances

- Cluster legend:
  - List every detected cluster with a color swatch, a representative sample prompt, and its point count
  - Clicking an entry filters/highlights the view to that cluster
  - Clicking again clears the filter
  - Support toggling a visual outline (hull) around each cluster's spatial extent
  - Also show, per cluster, how many of the active operation's result variants (e.g. from a Consistency Check) belong to it
    - Include clusters that have no members among the currently loaded points, since a variant can belong to a cluster the loaded sample doesn't otherwise represent
- Axis legend (for operations that produce a labeled axis/direction):
  - Recolor every loaded point along a gradient based on its projection score onto the axis
  - Show the two human-readable poles of the axis next to the relevant score range
  - Show an optional LLM-generated name for the axis if the user requested one
  - Once a second (orthogonal) axis is available, its own gradient and pole labels replace the primary axis's when the user switches to it (see the color-mode toggle under Semantic axis projection below)
  - Goal: the axis is never shown as a bare, unlabeled number

## Per-operation configuration UI

A dropdown lets the user choose which navigation operation is currently active; only that operation's own controls should be shown below it, each with only the controls that operation actually needs:

- **Inspection (no-op)**:
  - Controls: none operation-specific
  - Requires: a selection of one or more points to inspect
- **Nearest neighbor**:
  - Controls: a neighbor-count control
  - Requires: at least one selected point
- **Linear interpolation (lerp) / spherical interpolation (slerp)**:
  - Controls: an editable alpha schedule (the interpolation steps between the two endpoints) with quick-fill presets (e.g. evenly spaced 3/5/10 steps in [0,1])
  - Controls: an opt-in toggle for whether each generated trajectory step should also look up its nearest real dataset prompt
  - Requires: exactly two selected points
- **Direction/delta walk**:
  - Controls: its own alpha-schedule presets in the range [-2,4] (e.g. negative, forward, and full-range presets)
  - Controls: show the three selected points with their role labels (direction-from / direction-to / walk-start) once all three are selected
  - Controls: the same opt-in "nearest real dataset prompt per step" toggle as lerp/slerp
  - Requires: exactly three selected points
- **Semantic axis projection**:
  - Controls: two disjoint point groups ("Group A" / "Group B"), each with an "add selected points to this group" action and a "clear group" action, keeping the two groups mutually exclusive
  - Controls: an optional "compute a human-readable label for this axis" toggle (note in the UI that enabling it adds extra processing time)
  - Controls: once a second (orthogonal) axis is available, a toggle to switch which axis's scores currently drive the point coloring/display
  - Requires: at least one point in each group
- **Consistency check**:
  - Controls: a variant-count control and a variation-strength control, with inline guidance distinguishing a "light word-level edit" regime at low strength from a "full LLM paraphrase" regime at high strength
  - Requires: exactly one selected point (the seed)
- **Latent-space probing**:
  - Controls: a "place a probe point" mode that lets the user click a position directly on the graph instead of selecting existing points
  - Controls: once placed, let the user fine-tune the probe position via direct numeric coordinate input instead of only re-clicking
  - Controls: an optional "reconstruct text for this position" toggle that unlocks further settings (a maximum search time, a maximum number of refinement iterations, and counts for how many nearby/neutral/dissimilar existing points to use as context)
  - Requires: a placed probe position rather than a point selection

Across all operations:

- Show a clear, specific warning whenever the current selection doesn't satisfy what the active operation requires, rather than only failing silently when the operation is run
- Document the selection interaction vocabulary (click / modifier-click / shift-drag / empty-click) directly in the UI as in-app help, not only in external documentation

## Known limitation to label, not engineer around

The graph should support a purely cosmetic idle animation (a slight, continuous jitter of point positions) to make the view feel alive and make orientation easier, but this means displayed positions are not always pixel-exact while it runs

- Provide a toggle to disable it
- Label its effect directly in the UI (e.g. "animate ON — positions not exact") so a smooth-looking view is never mistaken for a precise one
