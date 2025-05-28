---
marp: true
theme: default
paginate: true
footer: 'Network Analysis - PMF-UNIST 2024/2025'
---

# Applied Network Science: Case Study
## Analyzing the "Network of the Conclave"

Network Analysis - Lecture 11
Nikola Balic, Faculty of Natural Science, University of Split
Data Science and Engineering Master Program

---

## Learning Objectives

Today, we will:
1.  **Understand a Real-World Problem:** Deconstruct the "Network of the Conclave" roadmap.
2.  **Apply Network Science:** See how core concepts from this course are used to tackle a complex analytical challenge.
3.  **Step-by-Step Approach:** Walk through the process of data collection, network construction, analysis, and validation.
4.  **Connect Theory to Practice:** Bridge the gap between theoretical network science and a practical research project.
5.  **Appreciate the Workflow:** Understand the iterative nature of network-based analysis.

---

## The Challenge: "Network of the Conclave"

We're tasked with understanding and potentially replicating a model built by Soda-Iorio-Rizzo to analyze the network of cardinal-electors in a Papal Conclave.

**The Goal (from the problem description):**
"Not a prediction engine but a structured “heat-map” of who is most likely to aggregate votes when the conclave opens."

---

## Key Take-aways (from Soda-Iorio-Rizzo)

The Bocconi team combined three relationship layers among the 128 cardinal-electors—**institutional co-service, episcopal lineage, and informal affinities**—into a multiplex network.

They ranked cardinals on:
1.  *Status* (eigenvector centrality)
2.  *Information control* (betweenness centrality)
3.  *Coalition capacity* (custom index: clustering, degree, “bridging”)

Scores were adjusted by an age-term.
([Università Bocconi][1], [Aleteia][2])

---

## Why Network Science for This Problem? (1/2)

Network science is ideal because:
-   **Relationships are Key:** The problem is inherently about connections between cardinals. The entire system can be modeled as a set of entities (cardinals) and the relationships (various ties) between them.

---

## Why Network Science for This Problem? (2/2)

-   **Multiple Layers of Interaction:** Cardinals interact in various capacities (Curia, lineage, informal ties). This is perfect for *multilayer/multiplex network analysis*. (Recall Lectures on advanced network structures if covered, or introduce as a concept).
-   **Influence and Importance:** Concepts like *centrality* directly map to "status," "information control," and "coalition capacity." (Recall Lecture 3 on Metrics!).
-   **Structure Matters:** The overall structure of these relationships can reveal potential power dynamics and voting blocs. (Recall Lecture 5 on Communities!).

---

## Roadmap Step 1: Data Sources – The Building Blocks

The roadmap (Section 1) lists crucial data sources:

| Layer                   | Primary source                                                                          | What to scrape/download                                        |
| ----------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| Curia co-memberships    | Holy See’s directory of dicasteries ([Vatican][3])                                      | Roster of every prefect, secretary & member                    |
| Episcopal lineage       | Catholic-Hierarchy.org ([Catholic Hierarchy][4])                                        | XML/HTML pages for each cardinal’s consecrator chain           |
| Informal ties           | NCR, Crux, VaticanNews, etc. Example methodology note ([National Catholic Reporter][6]) | Mentions of mentorships, ideological blocs, travel delegations |
| Age & biographic fields | Aleteia age database ([Aleteia][2])                                                     | Birth year, nationality, theological orientation               |
| Optional extras         | “The-New-Pope” open dataset ([Zenodo][7])                                               | Betting odds, language skills, media visibility                |

---

## Data Sources – Network Science Lens

-   **Scope and Attributes:** This step defines the boundaries and characteristics of our network.
-   **Layers as Edge Sets:** Each "Layer" (Curia, Lineage, Informal) will form a distinct set of edges in our *multiplex network*.
-   **Node & Edge Information:** "What to scrape/download" defines potential **nodes** (Cardinals) and the **information** used to establish edges.
-   **Weights & Filtering:** "Tips" often guide decisions on **edge weighting** (e.g., current links higher) or **node/edge filtering** (e.g., electors < 80).

---

## Roadmap Step 2: Building the Multiplex Network

This is where we transform raw data into a structured network.

### 2.1 Data Cleaning – Essential Pre-processing

1.  **Normalize names/IDs:** Critical for consistent node identification across different data sources. A cardinal might be "John Smith" in one and "J. H. Smith" in another.
2.  **Time-stamp links:** Allows for *temporal network analysis* (Lecture 12) or weighting current links more heavily than past ones.
3.  **Prune non-electors:** Defines the final set of nodes in our network based on domain rules (age < 80 for this problem).

**Network Science Lens:** Ensures data quality for an accurate network representation. Node and edge definitions are refined here.

---

### 2.2 Layer Construction – Defining Relationships

The roadmap details how to construct each layer:

| Layer    | Nodes     | Edges                               | Edge weight suggestion       |
| -------- | --------- | ----------------------------------- | ---------------------------- |
| Curia    | cardinals | shared dicastery ≥ 6 mo             | 1 per shared body            |
| Lineage  | cardinals | direct consecrator / co-consecrator | 1 (directed)                 |
| Informal | cardinals | coded tie                           | 1 if confirmed by ≥2 sources |

---

### Layer Construction – Network Science Lens (1/2)

-   **Nodes:** Consistently 'cardinals' across all layers.
-   **Edges:** Represent specific relationships.
    -   *Curia:* Undirected, weighted by number of shared bodies.
    -   *Lineage:* **Directed** graph. A cardinal consecrates another. (Recall Lecture 2 on Graph Types).
    -   *Informal:* Undirected, weighted by source confirmation.
-   **Edge Weights:** Quantify the strength or multiplicity of ties.

---

### Layer Construction – Network Science Lens (2/2)

-   **Implementation:** Python (NetworkX) or R (igraph/tidygraph) are standard tools. (Recall Lecture 1 / Tools Lecture).
-   **Aggregation Strategy:**
    -   The layers are combined into an *aggregated weighted graph*.
    -   `total_weight = w₁Curia + w₂Lineage + w₃Informal`
    -   This is a common technique in *multiplex network analysis* (Lecture 13).
    -   Initially, equal weights (w₁, w₂, w₃) are suggested, with sensitivity tests performed later.

---

## Roadmap Step 3: Network Analysis – The Three Pillars

Applying network metrics (Lecture 3!) to quantify influence.

| Pillar       | Metric (igraph/NetworkX)                            | Normalisation | Comment                                                     |
| ------------ | --------------------------------------------------- | ------------- | ----------------------------------------------------------- |
| Status       | `eigenvector_centrality()`                          | z-score       | Works best on connected graph—keep GC only                  |
| Info-control | `betweenness_centrality(normalized=True)`           | z-score       | Use edge weights = 1/totalWeight so busy ties shorten paths |
| Coalition    | ⅓ \* clustering + ⅓ \* degree + ⅓ \* “bridgeness”\* | min–max 0-1   | *Bridgeness = 1 – edge-constraint (Burt)*                   |

---

### Pillars – Network Science Lens: Status

-   **Pillar: Status**
-   **Metric: Eigenvector Centrality** `eigenvector_centrality()`
    -   Measures influence by connection to other influential nodes. A node is important if its neighbors are important.
    -   **Comment:** "Works best on connected graph—keep GC only." This means the analysis should focus on the Giant Component (Lecture 4 on Connectivity), as eigenvector centrality is ill-defined or less meaningful in disconnected graphs.
    -   **Normalisation:** z-score, to standardize scores.

---

### Pillars – Network Science Lens: Info-control

-   **Pillar: Information Control**
-   **Metric: Betweenness Centrality** `betweenness_centrality(normalized=True)`
    -   Identifies nodes that lie on many shortest paths between other pairs of nodes, thus controlling information flow.
    -   **Comment:** "Use edge weights = 1/totalWeight so busy ties shorten paths." This is a crucial modeling choice. Stronger ties (higher original weight) should make paths "shorter" in terms of influence flow, so their inverse is used.
    -   **Normalisation:** z-score.

---

### Pillars – Network Science Lens: Coalition

-   **Pillar: Coalition Capacity**
-   **Metric: Custom Index** (⅓ \* clustering + ⅓ \* degree + ⅓ \* “bridgeness”)
    -   *Clustering Coefficient:* Measures the tendency of a node’s neighbors to be connected to each other, indicating tight-knit local groups. (Lecture 3 & 5).
    -   *Degree Centrality:* Simple count of direct connections.
    -   *"Bridgeness" (1 – Burt's Constraint):* Measures how much a node connects otherwise disconnected parts. High bridgeness (low constraint) means the node is a bridge, essential for coalition building.
    -   **Normalisation:** min–max 0-1.

---

### Age Adjustment – A Domain-Specific Heuristic

`AdjScore = RawScore × exp(-|age – 66| / 15)`

-   This is not a pure network metric but a heuristic based on historical data (papal election ages).
-   It boosts scores for cardinals near the historical mean papal age (≈66) and discounts scores for those much younger or older.
-   The value '15' likely represents an estimated standard deviation or a scaling factor.
-   Common in applied modeling to incorporate domain knowledge.

---

## Roadmap Step 4: Rankings & Visualisations

Communicating the complex findings effectively.

1.  **Rank** cardinals based on each pillar and calculate an overall percentile.
2.  Create a **polar plot (radar chart)** for the top-15 `papabili` to compare their scores across the three pillars.
3.  Export the network to a **Gephi .gexf file** for interactive exploration by non-technical users.
4.  Produce a **snapshot HTML report** including network diagrams.
    -   Example visualization: Node size = Eigenvector Centrality, Node color = Continent.
    -   Layout algorithm: ForceAtlas2 (common in Gephi for revealing community structures).

---

### Rankings & Visualisations – Network Science Lens

-   **Importance of Visualization:** Crucial for understanding network structure, identifying key players, and conveying complex information simply.
-   **Mapping Metrics to Visuals:** Mapping network metrics (like centrality scores) to visual properties (node size, node color, edge thickness) is a standard and powerful technique.
-   **Interactive Exploration:** Tools like Gephi allow for dynamic exploration, filtering, and changing visual mappings, which can lead to new insights.
-   **Layout Algorithms:** ForceAtlas2 is a force-directed layout that often reveals community structures by pulling connected nodes together and pushing apart unconnected ones. (Related to concepts in Lecture 5 on Communities).

---

## Roadmap Step 5: Validation & Robustness

How trustworthy and stable is the model's output?

| Approach             | How                                                       | Purpose                                          |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------ |
| Historical back-test | Rebuild 2005 & 2013 elector networks (data all available) | Did scores peak for Ratzinger & Bergoglio?       |
| Monte-Carlo nulls    | Randomly permute edges within degree sequence             | Are observed centralities statistically extreme? |
| Weight sensitivity   | Vary (w₁,w₂,w₃) in \[0.3–0.5] range                       | Check pillar ranking stability                   |

---

### Validation – Network Science Lens (1/2)

-   **Historical back-test:**
    -   A form of out-of-sample validation using past data.
    -   Tests if the model's "heat-map" would have highlighted actual historical outcomes.
    -   Common in predictive modeling and simulation.

-   **Monte-Carlo nulls (Configuration Model):**
    -   Compares observed network properties (e.g., centrality scores of top cardinals) against random networks that have the same degree sequence as the real network.
    -   This helps determine if the observed structure and resulting scores are statistically significant or if they could have arisen by chance given the degrees of the nodes. (Relates to concepts in Lecture 6 on Random Graphs and null models).

---

### Validation – Network Science Lens (2/2)

-   **Weight sensitivity analysis:**
    -   Tests the robustness of the findings (especially rankings) to the specific choices of weights (w₁, w₂, w₃) used in aggregating the multiplex layers.
    -   If rankings change dramatically with small changes in weights, the model's conclusions might be too dependent on these arbitrary choices.
    -   A stable model will show consistent high-ranking cardinals even when weights are varied within a reasonable range.

---

## Roadmap Step 6: Reproducibility Checklist

Ensuring others can understand, verify, and build upon your work.

1.  **Version-controlled repository** (e.g., GitHub) with a DOI (e.g., via Zenodo) for stable citation.
2.  **Environment file** (e.g., `environment.yml` for Conda, `renv.lock` for R) to specify software dependencies.
3.  **Clear codebook** for the "informal-tie" coding scheme, including inter-coder reliability (e.g., Cohen's kappa κ ≥ 0.7).
4.  **Computational notebooks** (Jupyter/R-Markdown) with a clear pipeline (e.g., knit-to-HTML).
5.  **Data sharing licences** and considerations for data privacy/ethics (e.g., derived adjacency lists instead of raw scrapes).

**Network Science Lens:** These are standard best practices in computational social science and data science, essential for transparency, credibility, and fostering collaborative research.

---

## Roadmap Step 7: Possible Extensions

Pushing the analysis further using more advanced network science techniques.

*   **Dynamic network analysis**: Time-slice edges to see how coalitions and influence evolve as new cardinals are created or relationships change. (Related to Lecture 12 on Temporal Networks).
*   **Text-embedding similarity**: Construct network edges based on the semantic similarity of texts (e.g., homilies, public statements) authored by cardinals.
*   **Agent-based voting simulation**: Model the conclave voting process as a dynamic process unfolding on the network, where centrality scores influence voting behavior. (Related to Lecture 10 on Dynamics on Networks).
*   **Public-data fusion**: Merge network-derived scores with external data like betting-market odds to identify where network structure might provide insights not captured by public sentiment.

---

## Roadmap Step 8: Ethical & Practical Caveats

Important considerations for responsible analysis and communication.

-   **Interpretation:** Network scores highlight *structural* prominence, not intrinsic merit (e.g., holiness, doctrinal correctness). This must be communicated clearly to avoid misinterpretation.
-   **Data Collection Ethics:** Respect website scraping policies (rate-limits, `robots.txt`) and terms of service.
-   **Model Limitations:** Remember George Box: "All models are wrong, but some are useful." The model is a simplified representation of reality, a tool for understanding, not an absolute predictor of truth.

**Network Science Lens:** Ethical application of network analysis and transparent communication of its limitations are paramount.

---

## Project Quick Start Timeline (from Roadmap)

A practical, phased approach:

| Week | Milestone                                     | Network Science Focus                   |
| ---- | --------------------------------------------- | --------------------------------------- |
| 1-2  | Scraping + master ID reconciliation           | Data collection, Node definition        |
| 3-4  | Multiplex build + metric scripts              | Edge definition, Weighting, Centrality  |
| 5    | Validation runs + sensitivity analysis        | Null models, Robustness checks          |
| 6    | Visuals, narrative report, repository release | Communication, Reproducibility          |

This timeline integrates network science tasks into a manageable project plan.

---

## Connecting to Our Course (1/2)

This case study draws upon many concepts:

-   **L02: Graph Fundamentals:** Node/edge definitions, directed/undirected graphs, weights.
-   **L03: Network Metrics:** Centrality (degree, eigenvector, betweenness), clustering.
-   **L04: Connectivity:** Giant Component analysis for eigenvector centrality.
-   **L05: Community Detection:** Implicit in "coalition capacity" (clustering) and understanding voting blocs.

---

## Connecting to Our Course (2/2)

-   **L06: Random Graph Models:** Used in Monte-Carlo null model validation (configuration model).
-   **L10: Dynamics on Networks:** Relevant for extensions like agent-based voting simulations.
-   **L12: Temporal Networks (Future):** Directly applicable for dynamic network analysis extension.
-   **L13: Multilayer Networks (Future):** The core of this model is a multiplex/multilayer network.

---

## Student Task / Discussion Point

Consider **ONE** of the data layers (Curia, Episcopal, or Informal).

**For your chosen layer:**

---

### Discussion Point 1: Data Challenges

What are the primary challenges you anticipate in **collecting and cleaning data** for your chosen layer?

*Think about:*
    -   Data format (structured, unstructured?)
    -   Consistency across sources
    -   Missing data
    -   Subjectivity (for informal ties)

---

### Discussion Point 2: Edge Definition & Impact

How might your choice of **edge definition or weighting** for this layer significantly impact the resulting network structure and centrality scores?

*Think about:*
    -   Thresholds for creating an edge (e.g., "shared dicastery ≥ 6 mo")
    -   Binary vs. weighted edges
    -   Directionality

---

### Discussion Point 3: Layer-Specific Extension

If you were to extend the analysis of this *single layer*, what additional network science question might you ask, and what metric or technique could help answer it?

*Think about:*
    -   Community detection within that layer
    -   Identifying critical nodes/edges specific to that layer's function
    -   Temporal changes if applicable

---

## Conclusion (1/2)

The "Network of the Conclave" roadmap is an excellent example of an applied network science project. It involves:
-   **Data-driven network construction** from diverse, real-world sources.
-   Creation of a **multiplex network** to capture different types of relationships.
-   Application of core **network metrics** (centralities, clustering) to answer domain-specific questions about influence and coalition potential.

---

## Conclusion (2/2)

-   Rigorous **validation techniques** (back-testing, null models, sensitivity analysis) to assess model trustworthiness.
-   Emphasis on **reproducibility** and ethical considerations.
-   Clear pathways for **extensions** using more advanced network science methods.

This step-by-step approach, grounded in network science principles, allows for a structured and insightful analysis of a complex social system.

---

## Next Steps

-   If this were a course project, you would now begin the data collection and cleaning phase (Weeks 1-2).
-   Consider how the tools and techniques discussed in previous lectures (and future ones on specific tools) would be applied at each step.

**Next Lecture Topic (as per schedule):** E.g., Network Analysis Tools / Temporal Networks.

[1]: https://www.unibocconi.it/en/news/network-conclave?utm_source=chatgpt.com "In the Network of the Conclave - Bocconi University"
[2]: https://aleteia.org/2025/03/13/francis-nearing-average-length-of-pontificates-since-1800?utm_source=chatgpt.com "Francis nearing average length of pontificates since 1800 - Aleteia"
[3]: https://www.vatican.va/content/romancuria/en.html?utm_source=chatgpt.com "The Roman Curia - The Holy See"
[4]: https://www.catholic-hierarchy.org/?utm_source=chatgpt.com "Catholic-Hierarchy: Its Bishops and Dioceses, Current and Past"
[5]: https://en.wikipedia.org/wiki/Papal_conclave?utm_source=chatgpt.com "Papal conclave - Wikipedia"
[6]: https://www.ncronline.org/vatican/how-pope-francis-holds-powerful-sway-over-2025-cardinal-electors?utm_source=chatgpt.com "How Pope Francis holds powerful sway over the 2025 conclave"
[7]: https://zenodo.org/records/15383307?utm_source=chatgpt.com "The New Pope: Data Science Analysis of the 2025 Papal Conclave"
[8]: https://www.the-new-pope.org/index.html?utm_source=chatgpt.com "Conclave Network: The New Pope"
[9]: https://en.wikipedia.org/wiki/Eigenvector_centrality?utm_source=chatgpt.com "Eigenvector centrality - Wikipedia"