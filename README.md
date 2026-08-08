# ArmorIQ Hardware Trojan Intelligence Platform

## What is ArmorIQ?

[ArmorIQ](https://armoriq.ai/) is a security-first platform for governing AI agents and MCP servers. Its platform focuses on controlling agent actions, enforcing intent-aware policies, providing visibility into agent/tool activity, and maintaining auditable security decisions.

For this project, the ArmorIQ concept is used to structure the Hardware Trojan detection workflow around:

- **Specialized agents**
- **MCP-style tools/servers**
- **AI-based inference**
- **Policy/orchestration logic**
- **Monitoring and reporting**
- **Auditable detection results**

This makes the Hardware Trojan detector more modular than a standalone ML classifier.

---

## Project Objective

The objective is to identify potentially malicious or anomalous logic embedded inside RTL designs before hardware fabrication.

The system attempts to detect characteristics associated with Hardware Trojans, including:

- Suspicious trigger logic
- Unusual signal structures
- Abnormal fan-in/fan-out
- Rare or isolated signals
- Suspicious naming patterns
- Unusual signal widths
- Complex control logic
- Potential payload structures
- Deviations from a golden/reference design

The final detector can operate in three modes:

1. **Hybrid — GNN + Statistical**
2. **GNN Only**
3. **Statistical Only**

---

# System Architecture

```text
                    Verilog RTL Files
                           │
                           ▼
                  ┌─────────────────┐
                  │   RTL Parser    │
                  │                 │
                  │ Signals         │
                  │ Assignments     │
                  │ Always Blocks   │
                  │ Instances       │
                  │ Parameters      │
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Graph Builder   │
                  │                 │
                  │ Nodes = Signals │
                  │ Edges = Data    │
                  │         Flow    │
                  └────────┬────────┘
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
        ┌────────────────┐   ┌──────────────────┐
        │   GNN / GAT    │   │ Statistical      │
        │    Detector    │   │ Detector         │
        │                │   │                  │
        │ 4-layer GAT    │   │ Structural       │
        │ 256 hidden     │   │ anomalies        │
        │ attention      │   │ patterns         │
        └───────┬────────┘   └────────┬─────────┘
                │                     │
                │     Scores          │
                └──────────┬──────────┘
                           ▼
                 ┌─────────────────────┐
                 │  Hybrid Detector    │
                 │                     │
                 │ 60% GNN             │
                 │ 40% Statistical     │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Detection Agent     │
                 │                     │
                 │ HT-Free /           │
                 │ HT-Infested         │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Analysis Agent      │
                 │                     │
                 │ Trojan type         │
                 │ Fingerprint         │
                 │ Structural metrics  │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Monitor Agent       │
                 │                     │
                 │ Alerts              │
                 │ Dashboard metrics   │
                 │ Threat feed         │
                 └──────────┬──────────┘
                            │
                            ▼
                 JSON / CSV / Dashboard
```

---

# Core Components

## 1. Verilog RTL Parser

The `CompetitionVerilogParser` converts uploaded Verilog code into a structured `VerilogModule` representation.

It extracts:

- Module name
- Input signals
- Output signals
- Wires
- Registers
- Inout signals
- Signal widths
- Clock/reset indicators
- Assignments
- `always` blocks
- Module instances
- Parameters

Comments are removed before parsing to reduce noise.

This structured representation becomes the input for the graph and statistical analysis stages.

---

## 2. Graph Representation of RTL

The project represents RTL as a graph using **PyTorch Geometric**.

### Graph structure

- **Nodes:** Verilog signals
- **Edges:** relationships/data-flow between signals
- **Node features:** structural, semantic, and graph metrics
- **Edge attributes:** connection information

The graph builder calculates features such as:

- Signal type
- Signal width
- Clock/reset indicators
- Fan-in
- Fan-out
- Betweenness centrality
- Closeness centrality
- PageRank
- Clustering coefficient
- Isolation indicators
- High fan-in/fan-out indicators
- Suspicious naming patterns

The implementation uses a **48-dimensional node feature representation**.

---

# 3. ArmorIQ GNN Model

The main deep-learning component is `ArmorIQ_GNN`.

The model uses a **4-layer Graph Attention Network (GAT)** with:

- 48 input features
- 256 hidden dimensions
- 8 attention heads per GAT layer
- Batch normalization
- ELU activation
- Dropout
- Residual connections
- Global mean pooling
- Global max pooling
- Global sum pooling
- Fully connected classification head

### Model flow

```text
48-D Node Features
        │
        ▼
Linear Embedding
        │
        ▼
4 × Graph Attention Layers
        │
        ▼
BatchNorm + ELU + Residual Connections
        │
        ▼
Mean + Max + Sum Graph Pooling
        │
        ▼
Fully Connected Classifier
        │
        ▼
2-Class Output
        │
        ├── HT-Free
        └── HT-Infested
```

The GNN produces both a binary prediction and a probability score representing the estimated Trojan likelihood.

---

# 4. Statistical Trojan Detector

The statistical detector provides a second, independent signal.

It evaluates structural anomalies in the RTL rather than relying only on learned graph representations.

### Structural metrics

The detector considers:

- Number of signals
- Number of assignments
- Number of `always` blocks
- Number of instances
- Input/output ratios
- Register ratio
- Average fan-in
- Maximum fan-in
- Average fan-out
- Maximum fan-out
- Average signal width
- Maximum signal width
- Logic complexity

### Suspicious patterns

It also checks for suspicious signal names and structures related to terms such as:

- `trigger`
- `payload`
- `secret`
- `hidden`
- `attack`
- `leak`
- `covert`
- `kill`
- `backdoor`
- `bypass`
- `shadow`
- `ghost`

Additional anomaly checks include:

- Unusual widths
- High fan-out
- Isolated signals
- Rare signals
- Complex logic
- Golden-model deviations

The statistical analyzer produces an anomaly score and confidence estimate.

---

# 5. Hybrid Detection Model

The main detection system combines the GNN and statistical detector.

By default:

```text
GNN Weight          = 0.6
Statistical Weight  = 0.4
```

The hybrid score is calculated as:

```text
Hybrid Score =
    0.6 × GNN Score +
    0.4 × Statistical Score
```

The final classification is:

```text
Hybrid Score > 0.5  →  HT-Infested
Hybrid Score ≤ 0.5  →  HT-Free
```

The system also returns:

- Hybrid score
- GNN score
- GNN confidence
- Statistical score
- Statistical confidence
- Final confidence
- Detected anomalies
- GNN embedding

This provides a more explainable result than relying on a single prediction source.

---

# 6. AI Agent Layer

The project contains three specialized agents.

## Detection Agent ⚡

The Detection Agent is responsible for the primary detection workflow.

It:

1. Starts GNN inference.
2. Requests statistical analysis.
3. Runs the hybrid detector.
4. Generates an HT-Free or HT-Infested verdict.
5. Assigns severity based on the hybrid score.
6. Commits the result to the classification layer.

The current thresholds are:

```text
Score > 0.75
    → High-alert HT-Infested

0.50 < Score ≤ 0.75
    → Warning-level HT-Infested

Score ≤ 0.50
    → HT-Free
```

---

## Analysis Agent 🔬

The Analysis Agent goes beyond binary detection.

It generates a structural fingerprint containing:

- Trojan type
- GNN score
- Statistical score
- Hybrid score
- Total signal bits
- Average fan-out
- Signal density
- Trojan status

It classifies potential Trojan behavior into categories such as:

- Combinational
- Sequential
- Functional
- Kill Switch
- Data Leakage
- Parametric

This helps explain **what kind of suspicious behavior** may be present.

---

## Monitor Agent 🛰️

The Monitor Agent aggregates detection results and produces dashboard-level security metrics.

It is responsible for:

- Threat monitoring
- Global alerts
- Detection statistics
- Critical findings
- Average confidence
- Design-level monitoring

This makes the system suitable for analyzing multiple RTL designs together.

---

# 7. MCP Server Architecture

The project defines an MCP-style registry containing eight specialized servers:

| Server | Purpose |
|---|---|
| `mcp-parse` | Verilog tokenization and RTL parsing |
| `mcp-graph` | RTL/netlist graph construction |
| `mcp-gnn` | GNN-based Trojan scoring |
| `mcp-stat` | Statistical anomaly detection |
| `mcp-golden` | Golden-model comparison |
| `mcp-classify` | Hybrid binary classification |
| `mcp-report` | Report generation |
| `mcp-monitor` | Threat monitoring and dashboarding |

The code records server calls, status, request counts, and heartbeat timestamps.

> The current implementation uses local simulated MCP calls to demonstrate the orchestration architecture. These calls are not equivalent to verified remote MCP requests to ArmorIQ infrastructure.

---

# 8. Golden Model Comparison

The application supports an optional **Golden Model Reference**.

A trusted reference design can be used to identify structural deviations between the analyzed design and an expected clean implementation.

This can help identify changes that may not be obvious from simple pattern matching.

The UI exposes a:

```text
Golden Model Reference
```

option which can be enabled during analysis.

---

# Hardware Trojan Taxonomy

The project recognizes multiple classes of Hardware Trojan behavior.

| Type | Description |
|---|---|
| Combinational | Trojan behavior triggered by combinational conditions |
| Sequential | Trojan behavior dependent on state or clock sequences |
| Functional | Modifies normal circuit functionality |
| Kill Switch | Disables or disrupts circuit operation |
| Data Leakage | Attempts to expose sensitive information |
| Parametric | Alters circuit characteristics or behavior |
| Covert Channel | Uses side-channel behavior for information leakage |

---

# Streamlit Dashboard

The complete application is implemented as an interactive Streamlit dashboard.

### Main interface capabilities

- Upload multiple Verilog RTL files
- Select detection engine
- Adjust GNN/statistical weights
- Enable golden-model analysis
- View RTL/netlist graphs
- View anomaly analysis
- View AI-agent logs
- View MCP server registry
- View Hardware Trojan taxonomy
- Compare detection scores
- Export results

Supported input files:

```text
.v
.vh
```

---

# Visualization

The dashboard provides visual analysis including:

### Netlist Graph

Displays the signal dependency graph and helps identify unusual connectivity.

### Threat Score

Shows:

- GNN score
- Statistical score
- Hybrid score

for each analyzed module.

### Method Comparison

Allows comparison between the GNN and statistical detection approaches.

### Agent Logs

Shows the sequence of actions performed by the Detection, Analysis, and Monitor agents.

---

# Reporting

The application can export analysis results in:

### JSON

Contains:

- Platform information
- Architecture details
- MCP call count
- Total designs
- Trojan count
- Clean count
- Critical count
- Average confidence
- Per-design predictions
- Hybrid/GNN/statistical scores
- Trojan types
- Signal counts
- Anomaly counts

### CSV

For multiple analyzed designs, the application can export tabular detection results for further analysis.

---

# Technologies Used

| Technology | Purpose |
|---|---|
| Python | Core implementation |
| Streamlit | Interactive dashboard |
| PyTorch | Deep learning |
| PyTorch Geometric | Graph Neural Networks |
| GATConv | Graph Attention Network |
| NetworkX | Graph construction and analysis |
| NumPy | Numerical processing |
| Pandas | Data processing and reporting |
| Plotly | Interactive visualizations |
| Regex | RTL parsing and pattern detection |
| MCP-style architecture | Tool/server orchestration |

---

# End-to-End Workflow

```text
1. Upload Verilog RTL
          ↓
2. Parse Verilog module
          ↓
3. Extract signals and assignments
          ↓
4. Build RTL dependency graph
          ↓
5. Generate 48-dimensional graph features
          ↓
6. Run GNN inference
          ↓
7. Run statistical anomaly analysis
          ↓
8. Optionally compare against golden reference
          ↓
9. Combine GNN + statistical scores
          ↓
10. Detection Agent generates verdict
          ↓
11. Analysis Agent creates Trojan fingerprint
          ↓
12. Monitor Agent updates security metrics
          ↓
13. Display results
          ↓
14. Export JSON / CSV report
```

---

# Running the Project

## 1. Clone the repository

```bash
git clone https://github.com/Jiya404/Armoriq
cd https://github.com/Jiya404/Armoriq
```

## 2. Install dependencies

```bash
pip install -r requirements.txt
```

## 3. Start the Streamlit application

```bash
streamlit run app.py
```

Then open the local Streamlit URL shown in the terminal.

---

# Example Use Case

Upload a Verilog RTL design containing a suspicious trigger or payload.

The system:

```text
RTL
 ↓
Parser
 ↓
Graph representation
 ↓
48-D node features
 ↓
GAT inference
 ↓
Statistical anomaly detection
 ↓
Hybrid score
 ↓
Trojan verdict
 ↓
Trojan fingerprint
```

The dashboard can then show whether the design is classified as:

```text
HT-FREE
```

or

```text
HT-INFESTED
```

along with confidence, anomaly information, graph structure, and supporting metrics.


# Current Scope and Limitations

This repository is a research/prototype implementation.

In particular:

- The GNN architecture is defined in the application, but this file does not itself demonstrate a complete training pipeline or a loaded production checkpoint.
- MCP server interactions are implemented as local simulated calls for orchestration and UI demonstration.
- The RTL parser is regex-based and therefore does not replace a complete Verilog/SystemVerilog compiler.
- Detection quality depends on the quality and training state of the underlying GNN and on the statistical heuristics.
- Golden-model comparison is optional and depends on an appropriate reference representation.

For production deployment, the next steps would include trained and versioned model checkpoints, a larger labeled RTL/HT dataset, formal RTL parsing, real MCP/ArmorIQ integration, and systematic benchmark evaluation.

---


## Credits & References

- [ArmorIQ](https://armoriq.ai/) — AI-agent security and governance platform
- [ArmorIQ Platform Documentation](https://docs.armoriq.ai/platform)
- PyTorch
- PyTorch Geometric
- Streamlit
- NetworkX
- Plotly

---

## License

This project is intended for research and educational purposes.

