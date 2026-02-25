---
layout: post
title: "Building a Nuke Gizmo Manager in Python"
date: 2026-02-25
category: Pipeline
tags: [Nuke, Python, Pipeline]
description: "A walkthrough of building a production-ready gizmo manager panel for Foundry Nuke, covering custom PySide2 widgets, gizmo discovery, and versioning."
thumbnail: "https://picsum.photos/seed/nukegizmo/1200/675"
---

Writing pipeline tools for Nuke sits in an interesting space — you're writing software for artists who need it to be invisible when it works and obvious when it doesn't. This post walks through a gizmo manager I built to handle cross-project gizmo discovery, versioning, and one-click installation for a small studio pipeline.

## The Problem

Studios accumulate gizmos. Tools get dropped into shared drives, versioned inconsistently, and artists end up copy-pasting `.nk` files by hand. A gizmo manager solves this by providing a central registry and a panel UI inside Nuke itself.

The requirements for this one were:

- Scan a network path for all `.gizmo` files recursively
- Group them by category (derived from subdirectory structure)
- Display version metadata from a sidecar `version.txt`
- Let artists install a gizmo into the current session with one click
- No external dependencies beyond what ships with Nuke

![A compositor's node graph in Foundry Nuke — the environment the gizmo manager lives inside.](https://picsum.photos/seed/nuke-graph/1200/500)

## Project Structure

```
pipeline/
├── gizmo_manager/
│   ├── __init__.py
│   ├── scanner.py       # File discovery logic
│   ├── registry.py      # In-memory registry with caching
│   ├── panel.py         # PySide2 panel definition
│   └── installer.py     # nuke.nodePaste wrapper
└── menu.py              # Nuke startup hook
```

## Scanning for Gizmos

The scanner walks the shared library path and builds a list of `GizmoEntry` dataclasses:

```python
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional

@dataclass
class GizmoEntry:
    name: str
    path: Path
    category: str
    version: Optional[str] = None
    description: Optional[str] = None

def scan_library(root: Path) -> list[GizmoEntry]:
    entries = []
    for gizmo_path in sorted(root.rglob("*.gizmo")):
        relative = gizmo_path.relative_to(root)
        # Category is derived from the immediate parent directory name
        category = relative.parts[0] if len(relative.parts) > 1 else "Uncategorized"
        version = _read_version(gizmo_path.parent)
        entries.append(GizmoEntry(
            name=gizmo_path.stem,
            path=gizmo_path,
            category=category,
            version=version,
        ))
    return entries

def _read_version(directory: Path) -> Optional[str]:
    version_file = directory / "version.txt"
    if version_file.exists():
        return version_file.read_text().strip()
    return None
```

The key choice here is using `pathlib` throughout rather than `os.path`. Nuke runs Python 3.9+ in recent versions, so this is safe.

## The Registry

A simple in-memory registry wraps the scanner and adds caching to avoid re-scanning on every panel open:

```python
import time
from pathlib import Path

class GizmoRegistry:
    _CACHE_TTL = 60  # seconds

    def __init__(self, library_root: Path):
        self._root = library_root
        self._entries: list[GizmoEntry] = []
        self._last_scan: float = 0.0

    def get_all(self) -> list[GizmoEntry]:
        if time.time() - self._last_scan > self._CACHE_TTL:
            self._refresh()
        return self._entries

    def get_by_category(self) -> dict[str, list[GizmoEntry]]:
        result: dict[str, list[GizmoEntry]] = {}
        for entry in self.get_all():
            result.setdefault(entry.category, []).append(entry)
        return result

    def _refresh(self) -> None:
        self._entries = scan_library(self._root)
        self._last_scan = time.time()
```

## Installing a Gizmo

Installation is a thin wrapper around `nuke.nodePaste`, which reads a `.nk` or `.gizmo` file and adds the node to the current DAG:

```python
import nuke

def install_gizmo(entry: GizmoEntry) -> None:
    """Load a gizmo into the current Nuke session."""
    path_str = str(entry.path).replace("\\", "/")
    try:
        nuke.nodePaste(path_str)
    except RuntimeError as exc:
        nuke.message(f"Failed to load {entry.name}:\n{exc}")
```

The `replace("\\", "/")` is not optional on Windows — Nuke's file path handling is inconsistent with backslashes even in recent versions.

![The finished gizmo manager panel docked inside Nuke's pane layout, showing the tree view with categories expanded.](https://picsum.photos/seed/pyside-panel/1200/500)

## Panel UI

The panel uses PySide2, which ships with Nuke. The tree structure maps naturally to `QTreeWidget` with top-level items as categories and child items as gizmos:

```python
from PySide2 import QtWidgets, QtCore

class GizmoManagerPanel(QtWidgets.QWidget):
    def __init__(self, registry: GizmoRegistry):
        super().__init__()
        self._registry = registry
        self._setup_ui()

    def _setup_ui(self) -> None:
        layout = QtWidgets.QVBoxLayout(self)
        layout.setContentsMargins(8, 8, 8, 8)

        self._tree = QtWidgets.QTreeWidget()
        self._tree.setHeaderLabels(["Gizmo", "Version"])
        self._tree.setColumnWidth(0, 240)
        self._tree.itemDoubleClicked.connect(self._on_install)
        layout.addWidget(self._tree)

        refresh_btn = QtWidgets.QPushButton("Refresh")
        refresh_btn.clicked.connect(self._populate)
        layout.addWidget(refresh_btn)

        self._populate()

    def _populate(self) -> None:
        self._tree.clear()
        categorized = self._registry.get_by_category()
        for category, entries in sorted(categorized.items()):
            parent = QtWidgets.QTreeWidgetItem([category, ""])
            parent.setFlags(parent.flags() & ~QtCore.Qt.ItemIsSelectable)
            for entry in entries:
                child = QtWidgets.QTreeWidgetItem([entry.name, entry.version or "—"])
                child.setData(0, QtCore.Qt.UserRole, entry)
                parent.addChild(child)
            self._tree.addTopLevelItem(parent)
            parent.setExpanded(True)

    def _on_install(self, item: QtWidgets.QTreeWidgetItem, _col: int) -> None:
        entry: GizmoEntry = item.data(0, QtCore.Qt.UserRole)
        if entry:
            install_gizmo(entry)
```

Storing the `GizmoEntry` as `UserRole` data on each tree item avoids maintaining a separate mapping and keeps the install handler clean.

## Registering the Panel

In `menu.py`, the panel gets registered with Nuke's panel system so it docks like any other panel:

```python
import nuke
import nukescripts
from pathlib import Path
from gizmo_manager.registry import GizmoRegistry
from gizmo_manager.panel import GizmoManagerPanel

LIBRARY_ROOT = Path("//studio-server/pipeline/gizmos")

_registry = GizmoRegistry(LIBRARY_ROOT)

def create_panel():
    return GizmoManagerPanel(_registry)

nukescripts.registerPanel(
    "com.studio.GizmoManager",
    create_panel,
)

nuke.menu("Pane").addCommand(
    "Gizmo Manager",
    lambda: nukescripts.restorePanel("com.studio.GizmoManager"),
)
```

The singleton `_registry` instance at module level means the cache persists across panel open/close cycles, which is the intended behavior.

## What I'd Do Differently

A few things I'd change if I were building this again:

- **Use a JSON manifest instead of scanning.** Filesystem traversal over a network share is slow and brittle. A `manifest.json` generated by a separate CI step would be faster and would allow richer metadata (author, preview thumbnail path, changelog).
- **Add a search field.** Once the library grows past ~50 gizmos the tree becomes unwieldy. A `QLineEdit` filtering the tree on input is a one-afternoon addition.
- **Handle Nuke version compatibility in the manifest.** Not all gizmos work across major Nuke versions. Tagging entries with a `nuke_min_version` field and graying out incompatible ones would save artist confusion.

The full source is a work in progress and lives in the studio's internal repository, but the patterns here should be transferable to most small-to-medium pipeline setups.
