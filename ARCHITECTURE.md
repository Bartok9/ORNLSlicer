# ORNLSlicer Architecture

## Purpose

This file is a current-state orientation map for developers and agents working in
ORNLSlicer. It explains where the main systems live, how data moves through the
application, and which seams are normally extended. It is not a replacement for
Doxygen comments, the user guide, or the migrated wiki content under `docs/`.

Use this document first when deciding where a change belongs. Use the source and
Doxygen-style comments for API details once the owning subsystem is clear.

## Repository Layout

- `include/`: public project headers, grouped by subsystem. Most cross-module
  type boundaries are visible here: managers, threading, geometry, G-code,
  graphics, widgets, windows, units, and step/path model types.
- `src/`: implementations matching the `include/` subsystem layout. Runtime
  entry, GUI wiring, slicing, parser, writer, loader, and OpenGL behavior live
  here.
- `resources/`: Qt resources and data embedded into the application. This
  includes icons, shaders, styles, generated config files, and canonical settings
  YAML in `resources/settings/`.
- `templates/`: user-facing printer/process template files installed with the
  application.
- `scripts/`: project utilities. `scripts/generate_master_config.py` generates
  the checked-in settings config artifacts from `resources/settings/*.yaml`.
- `docs/`: canonical prose documentation, including the user guide, migrated
  wiki pages, and contributor guides.
- `cmake/`: CMake helper modules for build metadata, git/version information,
  timestamps, and presets.
- `nix/`: Nix package definitions used by `flake.nix` for the application and
  packaged third-party dependencies.

## Runtime Entry Points

`src/main.cpp` is the runtime split between CLI and GUI behavior.

- All runs first enable desktop OpenGL and register Qt metatypes used by queued
  signals and slots. Important registered types include `QSharedPointer<Part>`,
  mesh pointers, nested `SegmentBase` vectors, unit types, `GcodeCommand`,
  `GcodeMeta`, `fifojson`, `nlohmann::json`, and `MeshLoader::MeshData`.
- When command-line arguments are present, the process creates a
  `QCoreApplication`, configures `QCommandLineParser` through
  `CommandLineConverter`, converts options into `SettingsBase`, and runs
  `MainControl`. `MainControl` loads models or a project, starts slicing through
  `SessionManager`, parses generated G-code when needed, and writes the final
  output file.
- With no extra arguments, the process creates a `QApplication`, applies the
  Fusion style, initializes Qt resources with `Q_INIT_RESOURCE(icons)`,
  `Q_INIT_RESOURCE(shaders)`, `Q_INIT_RESOURCE(styles)`, and
  `Q_INIT_RESOURCE(configs)`, then shows `MainWindow::getInstance()`.

The CLI and GUI paths share the same managers, settings model, slicers, writers,
and loaders. The main difference is which UI/controller object receives progress
signals and handles the final output.

## Core Data Flow

1. Import input geometry or a project:
   `SessionManager::loadModel()` uses `MeshLoader` for STL/mesh files, while
   `SessionManager::loadSession()` uses `SessionLoader` for saved projects.
   Loaded mesh data becomes `Part` instances stored by `SessionManager`.
2. Build the active settings context:
   `SettingsManager` loads the embedded master config, active global settings,
   templates, and layer/range overrides into `SettingsBase` objects. Parts also
   hold local `SettingsRange` data for layer-specific overrides.
3. Start slicing:
   `SessionManager::doSlice()` chooses the concrete slicer from
   `SlicerType` through `changeSlicer()`. Current concrete slicers are
   `PlanarSlicer`, `ImageSlicer`, `RadialSlicer`, and `HelicalSlicer`.
4. Preprocess geometry:
   The selected slicer prepares part steps from meshes, settings, and slicing
   mode. Planar, radial, helical, and image slicing each own their preprocessing
   and postprocessing rules under `src/threading/slicers/`.
5. Compute pathing:
   `TraditionalAST` queues dirty `Step` objects and dispatches them to
   `StepThread`. `Step` owns the layer or scan unit. `Layer` orders
   `IslandBase` instances. Islands own ordered `RegionBase` instances. Regions
   generate `Path` objects composed of geometry segments.
6. Write temporary G-code:
   `AbstractSlicingThread` selects a concrete `WriterBase` subclass from
   `GcodeSyntax`, then calls setup, per-step writing, and shutdown hooks.
   Segments delegate syntax-specific line, arc, travel, dwell, and region
   behavior to the selected writer.
7. Parse and visualize/export:
   `GCodeLoader` selects a parser, reads generated or imported G-code, creates
   visual segment layers, computes text coloring and layer metadata, and emits
   data to `GCodeWidget`, `GcodeBar`, `GCodeView`, `LayerTimesWindow`, and
   export controls. CLI mode uses the same parsing pass when it needs layer-time
   adjustment and output metadata before writing the final file.

## Major Subsystems

### Application Shell And UI

- `MainWindow` is the GUI singleton and top-level signal hub. It owns menus,
  docks, toolbars, the part view, G-code view, settings bar, layer bar, command
  output, slice dialog, and export dialogs.
- `widgets/part_widget/` manages part tree metadata and part-oriented controls.
  `PartWidget` connects `PartMetaModel`, `PartView`, toolbar actions, and
  selection/transform updates.
- `widgets/settings/` builds setting panes and rows from settings metadata.
  `SettingRowBase` handles dependency logic and writes user changes back to the
  active `SettingsBase` or selected range bases.
- `widgets/gcode*` and `windows/gcode_export.*` show parsed G-code text,
  visualization controls, segment/layer filtering, and export metadata.

### Session Management

- `SessionManager` is the central session singleton, exposed through `CSM`.
  It stores `Part` objects, raw model data used by project files, session
  history, current slicer state, and slice/export signals.
- `Part` is the model object for imported geometry. It owns root and sub-meshes,
  transformations, parent/child relationships, settings ranges, and generated
  step pairs.
- `SessionLoader` saves and loads project archives on a `QThread`. Project
  state includes global settings, session JSON, local ranges, model data, and
  version information.

### Settings And Config Generation

- `SettingsManager` is the settings singleton, exposed through `GSM`. It owns
  the master config, active global settings, template settings, layer-bar
  templates, console settings, and settings version checks.
- Canonical setting definitions live in `resources/settings/*.yaml`.
  `scripts/generate_master_config.py` produces
  `resources/configs/master.conf` and `resources/configs/setting_inputs.conf`.
- The generated configs are embedded through `resources/configs/configs.qrc`.
  `SettingsManager` loads them from `:/configs/master.conf` and
  `:/configs/setting_inputs.conf` at runtime.

### Geometry, Meshes, And Cross-Sectioning

- `geometry/mesh/` contains `MeshBase` plus closed/open mesh implementations,
  mesh factory helpers, faces, vertices, and CGAL/OpenMesh-backed utilities.
- `MeshLoader` uses Assimp and CGAL/OpenMesh types to load models, preserve raw
  model data for projects, and emit `MeshData` back to the session.
- `cross_section/` contains plane/mesh intersection and polygon stitching
  support used by slicers to turn meshes into layer geometry.
- `geometry/` path primitives (`Point`, `Polyline`, `Polygon`, `PolygonList`,
  `Path`, and segment types) are the shared data structures between slicing,
  path optimization, writing, and visualization.

### Slicing Threads And Path Model

- `AbstractSlicingThread` owns the slicer worker thread, cancellation flag,
  temporary output file, selected G-code syntax, selected writer, and the
  setup/write/shutdown G-code lifecycle.
- `TraditionalAST` implements the common threaded step queue for current slicers.
  It runs slicer-specific preprocessing, schedules dirty `Step` objects through
  `StepThread`, then runs postprocessing and G-code writing.
- `Step` is the abstract printable/scannable unit. `Layer`, `RadialLayer`,
  `HelicalLayer`, `ScanLayer`, and `GlobalLayer` specialize it.
- `IslandBase` groups regions for a layer. Current island families include
  polymer, support, raft, brim, skirt, laser scan, and thermal scan islands.
- `RegionBase` generates and optimizes paths for a specific region type such as
  perimeter, inset, skin, infill, skeleton, support, brim, skirt, raft, laser
  scan, or thermal scan.

### G-Code Parsers And Writers

- `gcode/writers/` contains `WriterBase` and concrete machine/syntax writers.
  `AbstractSlicingThread::setGcodeOutput()` is the central switch that maps
  `GcodeSyntax` to a writer and `GcodeMeta`.
- `gcode/parsers/` contains `ParserBase`, `CommonParser`, and concrete syntax
  parsers used by `GCodeLoader` to parse imported or generated files.
- `gcode/` shared types such as `GcodeCommand`, `GcodeMeta`, and motion
  estimation utilities are used by both parser and writer paths.

### OpenGL Visualization

- `BaseView` is the shared `QOpenGLWidget` base for camera handling, event
  routing, shader setup, render queues, and common view controls.
- `PartView` renders `PartObject` instances, printer objects, grid/axes,
  labels, slicing planes, overhangs, and seams from `PartMetaModel` updates.
- `GCodeView` renders parsed G-code through `GCodeObject`, segment filters,
  selected-line highlighting, ghosted part geometry, and printer context.
- `GraphicsObject` is the base drawable. Derived objects own their OpenGL
  buffers, transforms, parent/child draw relationships, and picking/collision
  data.

### Persistence And Loaders

- `MeshLoader`, `SessionLoader`, and `GCodeLoader` are `QThread` subclasses.
  They communicate back to managers and widgets with Qt signals.
- Project save/load, mesh import/reload/replace, G-code import, and generated
  G-code visualization are all async paths in GUI mode. Keep UI work on the GUI
  side of the signal boundary.

## Extension Points

- Add a setting by editing the correct YAML file under `resources/settings/`,
  regenerating `resources/configs/master.conf` and
  `resources/configs/setting_inputs.conf`, and updating code that reads the new
  key. Follow `docs/wiki/Adding-a-New-User-Setting.md` and
  `docs/wiki/Generating-the-Master-Settings-File.md`.
- Add a settings UI row by extending `widgets/settings/` and wiring the row type
  where `SettingTab` maps generated input metadata to concrete row widgets.
- Add a slicer by deriving from `TraditionalAST` or `AbstractSlicingThread`,
  implementing preprocessing, postprocessing, and G-code writing behavior,
  adding a `SlicerType`, and routing it through `SessionManager::changeSlicer()`.
- Add a G-code syntax by adding a `GcodeSyntax` value, `GcodeMeta`, a
  `WriterBase` subclass, any needed parser subclass, and the writer/parser
  selection switches in `AbstractSlicingThread` and `GCodeLoader`.
- Add a region or island by deriving from `RegionBase` or `IslandBase`, adding
  the matching enum/string mappings, constructing it during slicer preprocessing
  or layer additions, and ensuring its paths write through `WriterBase`.
- Add a graphics object by deriving from `GraphicsObject` or an existing object
  family, creating buffers and uniforms in the object, and adding it through the
  owning `BaseView` subclass (`PartView` or `GCodeView`).
- Add mesh or project IO by extending the threaded loader/saver path rather than
  doing blocking UI work. Mesh import belongs in `MeshLoader`/`MeshFactory`;
  project archive shape belongs in `SessionLoader` and `SessionManager`
  serialization helpers.

## Build And Generated Artifacts

- `CMakeLists.txt` builds an object library target named `ornlslicer_obj`, then
  links the main `ornlslicer` executable from it. On Windows it also creates an
  `ornlslicer_cli` executable. Source, header, and Qt resource files are found
  by recursive globs.
- Qt resource compilation is enabled with `CMAKE_AUTORCC`. Resources under
  `resources/**.qrc` are compiled into the application and initialized by the
  GUI entry path.
- Precompiled headers are controlled by `ORNLSLICER_ENABLE_PCH`; unity builds
  are controlled by `ORNLSLICER_ENABLE_UNITY_BUILD`.
- `ORNLSLICER_AUTO_GENERATE_MASTER_CONFIG` controls the CMake custom target that
  regenerates `master.conf` and `setting_inputs.conf` from settings YAML.
- `ORNLSLICER_BUILD_DOC` enables the Doxygen target configured from `Doxyfile`.
  Doxygen comments should remain near public APIs; this architecture document
  should stay at subsystem/navigation level.
- `flake.nix` provides the Nix package and `devShells.ornlslicerDev`. Package
  definitions for ORNLSlicer and bundled dependencies live under `nix/`.

## Developer Guardrails

- Treat `SessionManager`, `SettingsManager`, `PreferencesManager`, and
  `MainWindow` as singleton ownership hubs. Avoid creating parallel global state
  when a manager already owns the lifecycle.
- Respect Qt thread boundaries. Loader and slicer classes emit data and status
  across signals; GUI widgets should update UI state after those signals arrive,
  not from worker threads.
- Follow the local `QSharedPointer` ownership style for session, mesh, part,
  step, island, region, and graphics-object relationships. Raw pointers still
  exist in Qt-owned UI and some legacy code, but new shared model ownership
  should match nearby code.
- Do not edit `resources/configs/master.conf` or
  `resources/configs/setting_inputs.conf` by hand. Change
  `resources/settings/*.yaml`, run the generator, and commit the generated
  artifacts when settings metadata changes.
- Keep documentation layered: this file is the system map, `docs/` contains
  user/contributor workflow docs, migrated wiki pages contain task guidance, and
  Doxygen comments describe source APIs.
- For docs-only changes, a build is usually unnecessary. At minimum run
  `git diff --check` and verify new relative links resolve.
