# PCB Lab — Changelog

## v0.4.0 — Workspace Pro

- Rebuilt the workspace layout to be flexible and responsive instead of a fixed prototype screen.
- Added collapsible left library and right inspector panels.
- Added horizontal overflow-safe workspace tabs and toolbars so controls are not cut off on smaller screens.
- Changed component insertion to **click-to-place**: click a library part and it is added directly; no drag-and-drop is required.
- Added click-on-canvas positioning for selected schematic symbols and PCB footprints.
- Expanded workspaces: Schematic, PCB, Simulator, 3D, BOM, Fabrication, Rules, and Release.
- Added professional command groups: File, Edit, View, Place, Route, Inspect, Tools, Manufacture.
- Added richer Schematic toolset: Select, Place, Wire, Bus, Net label, Junction, No connect, Power, Measure, Annotate.
- Added richer PCB toolset: Select, Route, Via, Zone, Keepout, Dimension, Measure, Tune, Layer swap, Ratsnest.
- Added PCB layer visibility controls for F.Cu, B.Cu, F.Silk, Edge.Cuts and Ratsnest.
- Added Rules workspace for clearance, track width, via, edge and silkscreen constraints.
- Added Release workspace plus in-app patch history.
- Added browser local-save, project JSON export, and BOM CSV export.
- Added responsive breakpoints for desktop, laptop and narrow screens.

## v0.3.1 — Deploy Fix

- Added required SvelteKit app shell for Vercel builds.

## v0.3.0 — EDA Flow

- Initial Schematic, PCB, Simulator, 3D, BOM and Fabrication workspace flow.

---

### Engine status

The UI and interaction shell are being built toward a serious EDA workflow, but production-grade electrical simulation, Gerber/Excellon generation, real interactive routing geometry, and manufacturing-grade DRC still require dedicated engines. Those features must not be represented as complete until their actual engines are integrated and validated.
