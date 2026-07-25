# FPGA Routing Visualizer & Analyzer

**Proces razvoja informacionih sistema / Information Systems Development Process**

A Python tool for parsing, visualizing, and analyzing FPGA routing results produced by the [VPR](https://verilogtorouting.org/) place-and-route flow. It reads a Routing Resource Graph (RRG) and a routing solution, then renders an interactive 2-D grid with optional overlays for routing paths, wire congestion, bounding boxes, and HPWL (Half-Perimeter Wirelength) metrics.

---

## Table of Contents

- [Features](#features)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Getting Started](#getting-started)
- [Usage](#usage)
- [Visualization Modes](#visualization-modes)
- [Module Overview](#module-overview)
- [Data Files](#data-files)

---

## Features

| # | Feature |
|---|---------|
| 1 | **FPGA matrix visualization** – renders CLB blocks, IO pads, and routing channels on a 2-D grid |
| 2 | **Single-signal routing** – traces the complete path of one net through the CHANX/CHANY fabric |
| 3 | **Branching-factor routing** – shows all nets that fan out to exactly *k* sinks |
| 4 | **Wire congestion heatmap** – colours every wire segment by the number of signals sharing it |
| 5 | **Segment wire-usage overlay** – displays used/total wires per routing segment |
| 6 | **Bounding box visualization** – draws the axis-aligned bounding rectangle around a full route or just its terminals (SOURCE/SINK nodes) |
| 7 | **Top-N largest bounding boxes** – ranks and displays the nets with the biggest bounding boxes |
| 8 | **HPWL metric** – calculates and saves the Half-Perimeter Wirelength for every net |
| 9 | **Routing deviation analysis** – compares HPWL estimates against real wire counts (absolute and relative deviation) |
| 10 | **Bounding-box overlap heatmap** – visualizes how many terminal bounding boxes overlap each routing segment |
| 11 | **PNG export** – any visualization can be saved as a high-resolution PNG |

---

## Project Structure

```
fpga-routing-visualizer/
├── parse_all.py              # Entry point – interactive CLI menu
├── fpga_project/
│   ├── __init__.py
│   ├── models.py             # Data models: Node, Edge, RRG, Net, Route
│   ├── parser_rrg.py         # XML parser for the Routing Resource Graph
│   ├── parser_route.py       # Text parser for VPR .route files
│   ├── fpga_matrix.py        # Base class: draws the FPGA grid and maps RRG → grid coordinates
│   ├── fpga_routing.py       # Routing overlays (single net, branching, first-N nets)
│   ├── fpga_wires.py         # Wire congestion and segment wire-usage overlays
│   ├── fpga_bounding_box.py  # Bounding box metrics and visualizations
│   └── fpga_analysis.py      # HPWL computation, deviation analysis
└── b9/
    ├── rrg.xml               # Routing Resource Graph for the b9 benchmark
    ├── b9.route              # Final routing solution
    ├── iteration_001.route   # Per-iteration routing snapshots (001–060+)
    ├── b9.place              # VPR placement file
    ├── b9.blif               # BLIF netlist
    └── ...
```

The class hierarchy mirrors the feature layers:

```
FPGAMatrix
├── FPGARouting        (extends FPGAMatrix)
│   └── FPGABoundingBox  (extends FPGARouting)
│       └── FPGARoutingAnalysis  (extends FPGABoundingBox)
└── FPGAWires          (extends FPGAMatrix)
```

---

## Requirements

- Python 3.9+
- [matplotlib](https://matplotlib.org/)
- [bokeh](https://bokeh.org/) *(imported in `fpga_routing.py`)*

Install dependencies:

```bash
pip install matplotlib bokeh
```

---

## Getting Started

1. **Clone the repository**

   ```bash
   git clone https://github.com/aleksandarhorvat/fpga-routing-visualizer.git
   cd fpga-routing-visualizer
   ```

2. **Install dependencies**

   ```bash
   pip install matplotlib bokeh
   ```

3. **Run the interactive menu**

   ```bash
   python parse_all.py
   ```

The tool will prompt you to choose a route file (final or a specific iteration) and then present a numbered menu of visualization modes.

---

## Usage

```
$ python parse_all.py
Unesi broj rute, 0 ako je finalna: 0        ← 0 = final route; 1-060 = iteration snapshot

Izaberi prikaz:
 1 - Matrix
 2 - Routing jednog signala
 3 - Routing by branching
 4 - Wire congestion
 5 - Segment wire usage
 6 - Bounding box jednog signala
 7 - Terminal bounding box jednog signala
 8 - Najveći bounding box-ovi
 9 - Najveći terminal bounding box-ovi
10 - HPWL
11 - Prvih N signala
12 - Analiza odstupanja ruta od HPWL
13 - Vizualiacija broja preklapanja bounding box-ova na segmentima

Unesi broj prikaza: _
```

After each visualization you will be asked whether to save the result as a PNG file.

---

## Visualization Modes

| Mode | Description |
|------|-------------|
| **1 – Matrix** | Draws the raw FPGA grid (CLBs, IO pads, routing channels) with no routing overlay. |
| **2 – Routing jednog signala** | Prompts for a `net_id` and draws the complete routing path for that single net, including directional arrows and SOURCE/SINK labels. |
| **3 – Routing by branching** | Prompts for a branching factor *k*; shows all nets that reach exactly *k* SINK nodes, each in a distinct random colour. |
| **4 – Wire congestion** | Heatmap (Blues scale) showing how many distinct signals share each wire segment. |
| **5 – Segment wire usage** | Displays `used/total` wire counts per routing segment as a red-scale overlay. |
| **6 – Bounding box jednog signala** | Draws the bounding rectangle around all nodes of a selected net. |
| **7 – Terminal bounding box** | Draws the bounding rectangle around only the SOURCE and SINK nodes of a selected net. |
| **8 – Najveći bounding box-ovi** | Ranks all nets by full bounding-box area and draws the top *n*. |
| **9 – Najveći terminal bounding box-ovi** | Ranks all nets by terminal bounding-box area and draws the top *n*. |
| **10 – HPWL** | Calculates HPWL for every net and saves the results to `hpwl_metrika.txt`. |
| **11 – Prvih N signala** | Draws the first *n* nets in the route file, each in a different colour. |
| **12 – Analiza odstupanja** | Computes absolute and relative deviation between HPWL and real wire count; prints or saves top-N signals to `hpwl_odstupanje_analiza.txt`. |
| **13 – Preklapanja bounding box-ova** | Orange-scale heatmap showing how many terminal bounding boxes overlap each routing segment. |

---

## Module Overview

### `models.py`
Core data structures:
- **`Node`** – RRG node with id, type (`SOURCE`, `SINK`, `IPIN`, `OPIN`, `CHANX`, `CHANY`), PTC, and bounding coordinates.
- **`Edge`** – Directed edge between two nodes (source → sink).
- **`RRG`** – Container for all nodes and edges in the routing resource graph.
- **`Net`** – A single routed net with an id and an ordered list of `Node` objects.
- **`Route`** – Container mapping net serial numbers to `Net` objects.

### `parser_rrg.py` – `RRGParser`
Parses VPR's `rrg.xml` into an `RRG` object. Also provides helpers to look up the side (`TOP`, `BOTTOM`, `LEFT`, `RIGHT`) of a pin or wire node.

### `parser_route.py` – `RouteParser`
Parses VPR's plain-text `.route` files into a `Route` object.

### `fpga_matrix.py` – `FPGAMatrix`
Base visualization class. Renders the FPGA grid and builds `coord_map` – a dictionary mapping every RRG node id to its (x, y) position on the matplotlib canvas.

### `fpga_routing.py` – `FPGARouting`
Extends `FPGAMatrix`. Draws routed paths with directional arrows; supports single-net, branching-factor, and first-N-nets modes.

### `fpga_wires.py` – `FPGAWires`
Extends `FPGAMatrix`. Computes wire load per segment and renders congestion and usage heatmaps.

### `fpga_bounding_box.py` – `FPGABoundingBox`
Extends `FPGARouting`. Calculates axis-aligned bounding boxes (full route and terminal-only) and their area in cells; supports top-N ranking and bounding-box overlap heatmaps.

### `fpga_analysis.py` – `FPGARoutingAnalysis`
Extends `FPGABoundingBox`. Computes HPWL for all nets, real wire usage, and absolute/relative deviation metrics; saves results to text files.

---

## Data Files

| File | Description |
|------|-------------|
| `b9/rrg.xml` | Routing Resource Graph (XML) for the *b9* benchmark circuit |
| `b9/b9.route` | Final VPR routing solution |
| `b9/iteration_NNN.route` | Routing snapshot after iteration *NNN* (001–060+) |
| `b9/b9.place` | VPR placement file |
| `b9/b9.blif` | Technology-mapped BLIF netlist |
| `b9/pris_arch.xml` | FPGA architecture description |
| `hpwl_metrika.txt` | Output: HPWL values per net (generated by mode 10) |
| `hpwl_odstupanje_analiza.txt` | Output: routing deviation analysis (generated by mode 12) |
