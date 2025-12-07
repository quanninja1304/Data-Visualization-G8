# 🚀 News Popularity Analytics: The Data Engineering Strategy

> **Project Thesis:** Raw data is not an asset; it is a liability. Only through strategic transformation does it become intelligence.
>
> This project demonstrates the power of **Advanced Data Preparation** by transforming a hostile, fragmented dataset of **37.5 million records** into a high-precision strategic engine for predicting content virality.

## 🔎 Overview

In the modern attention economy, content creators face a dilemma: **How to cut through the noise?** The answer lies not in guessing, but in decoding the "physics" of information diffusion.

This project is a rigorous end-to-end data science implementation. We analyze the **["News Popularity in Multiple Social Media Platforms"](https://archive.ics.uci.edu/dataset/432/news+popularity+in+multiple+social+media+platforms)** dataset, a complex collection of social feedback logs spanning 48 hours for over 93,000 news articles.

### **The Core Challenge: "Analytically Hostile" Data**
In its raw state, this dataset actively prevents insight. It presents three fundamental barriers that disqualify it from standard analysis:
1.  **Fragmentation:** Insights are siloed across 13 separate physical files, making cross-platform benchmarking impossible.
2.  **Structural Unsuitability:** Time-series data is stored in a "Wide Format" (144 columns), preventing dynamic velocity analysis.
3.  **Integrity Decay:** Critical timestamps are stored as text, and popularity metrics are corrupted by placeholder values (`-1`).

**Our Solution:** We architected a **Local Map-Reduce Pipeline** to overcome these barriers. By engineering **15 new behavioral metrics** (including Velocity Vectors, Sentiment Divergence, and Market Saturation Scores), we transformed this raw chaos into a unified, actionable playbook for content strategy.

---

## 💾 The Dataset: Architecture & Anatomy

The dataset is a rare, granular log of news consumption, but its architecture is split into two distinct, disconnected layers.

### **Part A: The Metadata Layer (`News_Final.csv`)**

This is the "Anchor File" containing descriptive attributes for each of the **93,239 articles**. While it provides the core features, several columns require specific strategic handling to be useful.

| Feature Name | Data Type | Analytical Role & Strategic Handling |
| :--- | :--- | :--- |
| **`IDLink`** | `float` | **Primary Key.** The connector for joining all 13 files. <br>*Status:* Must be cast to `string` or `int` to prevent floating-point precision errors during merging. |
| **`Title`** <br> **`Headline`** | `object` | **Content DNA (Input).** Unstructured text strings representing the article's hook and summary. <br>*Status:* Analytically silent in raw form. We will apply **NLP (TextBlob)** to these columns to engineer our own `Sentiment` and `Complexity` features. |
| **`Source`** | `object` | **Context.** The publisher (e.g., 'nytimes'). <br>*Status:* Extremely high cardinality (5,700+ values) with messy variations. Requires **Entity Resolution** and grouping into **Source Tiers** (Mainstream vs. Niche). |
| **`Topic`** | `object` | **Category.** The subject matter (Economy, Microsoft, Obama, Palestine). Serves as our primary dimension for segmentation. |
| **`PublishDate`** | `object` | **Temporal Anchor.** The timestamp of publication. <br>*Status:* **Critical Failure.** Stored as a string. Must be converted to `datetime` to engineer `Hour`, `Day`, and `Market Saturation` metrics. |
| **`SentimentTitle`** <br> **`SentimentHeadline`** | `float` | **Black Box Metrics (IGNORED).** <br>*Status:* **We explicitly discard these columns.** <br>**Reasoning:** <br>1. **Unknown Methodology:** We do not know the algorithm used (e.g., VADER vs. NLTK). Different models weigh nuances like double negatives ("not bad") differently. <br>2. **Explainability:** To present a defensible story, we must use a **"White Box"** approach (TextBlob) where we can explain exactly how the score was derived. <br>3. **Consistency for Divergence:** To calculate **Sentiment Divergence** ($|Title - Headline|$), both texts must be scored by the *exact same model*. Using raw pre-computed values carries the risk of inconsistent baselines, rendering the divergence metric mathematically invalid. |
| **`Facebook`** <br> **`GooglePlus`** <br> **`LinkedIn`** | `int` | **Final Outcomes (Targets).** The total popularity score after 48 hours. <br>*Status:* <br>1. **Corrupted:** Uses `-1` as a placeholder for missing data (must be replaced with `NaN`). <br>2. **Skewed:** Follows a Power Law distribution (extreme outliers). Requires **Logarithmic Normalization** (`np.log1p`) for valid visualization. |

### **Part B: The Social Feedback Layer (12 Time-Series Files)**
These files contain the "Heartbeat" of the news—detailed, minute-by-minute popularity logs.
*   **File Structure:** `[Platform]_[Topic].csv` (e.g., `Facebook_Economy.csv`, `LinkedIn_Microsoft.csv`).
*   **Total Volume:** ~37.5 Million data points.

| Feature Name | Data Type | Analytical Role & The "Wide Format" Problem |
| :--- | :--- | :--- |
| **`IDLink`** | `int` | **Foreign Key.** Links back to the Metadata layer. |
| **`TS1` ... `TS144`** | `int` | **The Velocity Vector.** 144 columns representing sequential 20-minute time slices over a 48-hour period.<br><br>**The Structural Crisis:** This "Wide Format" is optimized for storage, not science. <br>• You cannot plot "Time" on an X-axis because Time is scattered across 144 columns.<br>• You cannot calculate "Velocity" (Rate of Change) efficiently.<br><br>**The Engineering Mandate:** We must perform a massive **Structural Transformation (`melt`)** to pivot this data into a "Long Format" (`ID` | `Time` | `Score`), unlocking the ability to model the lifecycle of virality. |

### **Summary of Data Health**
*   **Missing Values:** `-1` is used as a placeholder for "untracked," corrupting mean/sum calculations.
*   **Distribution:** Extreme Right-Skew. 90% of articles have near-zero engagement, while the top 1% have millions. Requires Logarithmic Normalization (`np.log1p`).
*   **Scale:** The integration of Part A and Part B results in a dataset that exceeds the RAM capacity of standard machines, necessitating a **Chunking Strategy**.

---

## 🛠 Engineering Pipeline: From Raw Logs to Strategic Assets

We transformed raw, fragmented logs into a high-dimensional strategic asset through a rigorous **Data Engineering Pipeline**. This process was architected to handle **Big Data (37.5M rows)** on local hardware without memory overflow.

### 1. Data Cleaning & Integration (The "Memory-First" Strategy)
*The raw data presented a critical RAM bottleneck (~ 144MB uncompressed across 13 "wide" files, which becomes 16GB after merging and consolidating), requiring a streaming-based approach.*

#### 🧩 Smart Integration ("Melt-then-Combine")
> **🔴 The Bottleneck:** Merging 13 files in their original "Wide Format" (144 columns) would create a massive intermediate sparse matrix, causing immediate Kernel Crash on standard 16GB RAM machines.
>
> **🟢 The Engineering Solution:** We implemented an **Iterative Map-Reduce Loop**:
> 1.  **Load** a single topic file.
> 2.  **Pivot** immediately to "Long Format" using `pd.melt()`, reducing column width to 3 (`ID`, `Time`, `Score`).
> 3.  **Append** to the master list and release memory.

#### 🧹 Placeholder Sanitization
* **Detection:** Identified statistical corruption where tracking errors were recorded as `-1`.
* **Action:** Replaced `-1` with `NaN`.
* **Impact:** Ensures aggregation functions (`mean`, `sum`) remain statistically valid, preventing "negative popularity" artifacts.

#### 🔗 Entity Resolution (Source Consolidation)
* **Problem:** The `Source` column contained **5,700+ noisy variations** (e.g., *'Reuters'*, *'Reuters via Yahoo'*, *'Reuters Online'*), fragmenting the analysis.
* **Optimization Trick ("Count on Small, Apply to Big"):**
    Instead of running expensive Regex on 37M rows, we extracted unique `(ID, Source)` pairs (~90k rows), cleaned them to create a Mapping Dictionary, and **broadcasted** the map back to the master dataset.
    * ⚡ **Performance Win:** Reduced processing time from **hours to seconds**.

### 2. Advanced Feature Engineering (The Strategic Engine)
We moved beyond simple view counts to engineer **15 Behavioral & Contextual Metrics**, categorized into four strategic dimensions.

#### **A. Dynamics (The Physics of Virality)**
*Treating time-series data as velocity vectors to measure "Force" and "Friction."*

* **`Initial_Velocity` ($V_0$):** Popularity score at `TimeSlice 1` (First 20 mins). Measures the **Impulse Power** (Click-Through Rate).
* **`Stickiness_Index` ($S$):** A dimensionless ratio measuring audience retention.
  
  $$S = 1 - \left( \frac{V_0}{\text{Final Score}} \right)$$
  
  * **High $S$ ($\to 1$):** Organic growth, "Evergreen" content.
  * **Low $S$ ($\to 0$):** High drop-off, "Flash" or "Clickbait" content.

#### **B. Psycholinguistics (Content DNA)**
*Using NLP to decode the psychological triggers in headlines.*

> **💡 Optimization Hack:**
> Running `TextBlob` sentiment analysis on 37M rows is computationally prohibitive.  
> **Our Approach:** We extracted unique Article Titles (~90k), ran the NLP pipeline once per title, and merged results back using `IDLink`.  
> **🚀 Speedup: ~400x.**  

* **`Sentiment_Divergence`:** The absolute difference between `Title_Sentiment` and `Headline_Sentiment`. Used to scientifically detect **Bait-and-Switch** tactics.
* **`Title_Complexity`:** A cognitive load metric (Word Count + Avg Word Length) to test the "Keep It Simple" hypothesis vs. "Value Signaling".

#### **C. Market Ecology (Context)**
* **`Source_Tier`:** Algorithmic classification of publishers into **Tier 1 (Mainstream Goliath)** vs. **Tier 3 (Niche David)** based on total historical volume.
* **`Opportunity_Score`:** A rolling-window metric measuring Market Saturation.
    * *Logic:* Inverse of the number of competing articles published within a $\pm 2$ hour window.
    * *Output:* Classifies time slots into **Red Ocean** (High Competition) ⚔️ vs. **Blue Ocean** (Low Competition) 🌊.

#### **D. Statistical Normalization**
* **Logarithmic Transformation:** Since the popularity data followed a strict **Pareto Distribution (Power Law)** (80/20 rule), we applied `np.log1p`:
    
    $$y = \log_{10}(x + 1)$$
    
    This compresses extreme outliers, revealing the "Hidden Majority" structure for visualization.
    
---

## 📊 Storytelling Key Findings

We structured our analysis into **4 Strategic Chapters**, using a **"Before/After"** framework to demonstrate how specific engineering steps unlocked business intelligence.

### **Chapter 1: Content DNA (Psycholinguistics)**
*Goal: Quantifying the economic impact of writing style.*

*   **Storyboard 1: The Clickbait Paradox**
    *   *The Engineering Unlock:* **Sentiment Divergence** ($|Title - Headline|$).
    *   *The Insight:* **Deception Kills Velocity.** High-divergence "Bait-and-Switch" headlines perform poorly on all platforms. The fastest-moving content ($V_0 > 100$) has low divergence ($< 0.20$), proving that **Tonal Alignment** drives impulse clicks.
*   **Storyboard 2: The Complexity Premium**
    *   *The Engineering Unlock:* **Cognitive Load Scoring** (Word Count + Avg Length).
    *   *The Insight:* **Detail Signals Value.** Disproving the "shorter is better" myth, we found a universal "Staircase Effect" where **Complex Titles** consistently outperform Simple ones by **+45%** on Facebook and **+30%** on LinkedIn.
*   **Storyboard 3: Topic Lifecycles**
    *   *The Engineering Unlock:* **Dynamic Lifecycle Vectors** (Time-Series Aggregation).
    *   *The Insight:* **Personality > Utility.** "Personality" topics (e.g., *Obama*) exhibit **Exponential Viral Growth** (Network Effects), while "Thematic" topics (e.g., *Economy*) hit a hard **Linear Utility Floor**.

### **Chapter 2: Market Ecology (Context)**
*Goal: Analyzing the competitive environment.*

*   **Storyboard 4: David vs. Goliath**
    *   *The Engineering Unlock:* **Source Tiering** (Tier 1 Mainstream vs. Tier 3 Niche).
    *   *The Insight:* **The Distribution Moat.** Tier 1 sources enjoy a **3x Performance Multiplier** over Tier 3 sources. Niche blogs hit a structural "Velocity Ceiling" ($V_0 \approx 50$), proving they cannot compete head-to-head on viral speed.
*   **Storyboard 5: The Red Ocean Paradox**
    *   *The Engineering Unlock:* **Opportunity Score** (Rolling Window Saturation).
    *   *The Insight:* **Traffic Trumps Competition.** Publishing during "Blue Ocean" windows (low competition) yields the **lowest** returns. Entering a "Red Ocean" (high saturation) yields an **8x Performance Premium** because supply follows aggregate audience demand.

### **Chapter 3: Engagement Dynamics (Virality Physics)**
*Goal: Mapping the unique "physics" of each platform.*

*   **Storyboard 6: Platform Archetypes**
    *   *The Engineering Unlock:* **Velocity ($V_0$) vs. Stickiness ($S$) Matrix.**
    *   *The Insight:* **Structural Inversion.**
        *   **Facebook** is a **Vertical Impulse Machine** (High Velocity, Low Retention). Strategy: *"Shock & Awe."*
        *   **LinkedIn** is a **Horizontal Utility Machine** (Low Velocity, High Stickiness). Strategy: *"Reference Value."*
*   **Storyboard 7: The "Sleeper Hit" Phenomenon**
    *   *The Engineering Unlock:* **The Stickiness Ratio** ($S \to 1.0$ with $V_0 \approx 0$).
    *   *The Insight:* **Zero Velocity $\neq$ Failure.** On professional networks, many top-performing articles had **zero initial views**. They survived via "Organic Discovery" (Search/SEO), validating the "Long Tail" strategy for niche content.

### **Chapter 4: Strategy Synthesis**
*Goal: The unified playbook.*

*   **Storyboard 8: The Golden Quadrant**
    *   *The Engineering Unlock:* **Multi-Vector Segmentation** (Intersection of Top Quartile $V_0$ & $S$).
    *   *The Insight:* **Viral Utility is an Establishment Game.** The "Holy Grail" zone (High Speed + High Retention) is gated by **Institutional Trust**. It is populated 86% by Tier 1/Tier 2 sources writing **Honest** (Low Divergence), **Complex** (High Load) headlines about **Personalities**.

---

## 🤖 Machine Learning Validation: The "Engineering Lift"

To scientifically prove that **data engineering** drives performance, we conducted a rigorous A/B experiment between a **Baseline Model** (Raw Data) and a **Strategic Model** (Engineered Data).

### **1. The Quantitative Verdict: 480% Performance Lift**

We compared **Linear Regression** on raw metadata against **XGBoost** on engineered features. The results confirm that raw metadata is practically useless without transformation.

| Performance Metric | Model A (The Lazy Baseline) | Model B (The Strategic Solution) | The "Lift" | Business Implication |
| :--- | :--- | :--- | :--- | :--- |
| **Input Data** | Raw ID, Timestamp, Counts | Tiers, Content DNA, Opportunity | **Quality** | Replaced "Volume" metrics with "Context" metrics. |
| **$R^2$ Score** | **0.043** (4.3%) | **0.250** (25.0%) | **+480%** | **Predictive Power.** Model A is guessing. Model B found the signal. |
| **MAE** (Mean Abs Error) | **180** points | **112** points | **-38%** | **Precision.** Model B is nearly 2x more accurate in predicting ROI. |
| **RMSE** (Root Mean Sq) | **564** | **580** | **+2.7%** | **Stability.** Model B gained accuracy without exploding on viral outliers. |

*   **Model A (Raw):** Failed due to the **Power Law Trap**. It could not reconcile the difference between a median article (6 views) and a viral hit (40k views), defaulting to a "safe average."
*   **Model B (Engineered):** Succeeded by using **Log-Transformation** ($y = \log(x+1)$) to linearize the viral curve and **Source Tiering** to capture authority.


### **2. Visualizing the Behavior**

Metrics ($R^2$) tell us *that* the model improved; visualization tells us *how*.

#### **Test A: The "Signal Detection" Check (Log-Log Scatter)**
*Does the model successfully distinguish between a Flop and a Hit?*

![Lift Comparison](figures/ml_models_lift_comparison.png)

*   **Model A (Left - Red):** Displays a **Flatline**. The model predicts a static range (~150 views) for *everyone*, regardless of actual success. It "gave up" on finding the signal.
*   **Model B (Right - Blue):** Displays a **Diagonal Trend**. As Actual Popularity rises, Predicted Popularity rises. The model successfully climbs the "Viral Ladder."

#### **Test B: The "Reality Check" (Density Distribution)**
*Does the model understand that most content fails?*

![Density Comparison](figures/ml_models_density_comparison.png)

*   **Model A (Red):** The **"Delusional Average."** The curve peaks around **316 points** (Log 2.5). It massively overestimates failure, predicting moderate success for every article.
*   **Model B (Blue):** The **"Reality Check."** The curve shifts left to **~5 points** (Log 0.7), aligning perfectly with the **Gray Truth Curve**.
*   `=>` **Implication:** Model B correctly identifies the "Long Tail" of low-performing content, achieving the **38% reduction in MAE**.

### **3. Forensic Analysis: The Top 10 Drivers of Virality**

We extracted the **Gain** (Feature Importance) from the winning XGBoost model to rank the factors that actually drive success.

| Rank | Feature Name | Category | Power (%) | Strategic Insight |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Topic_Obama** | Context (Subject) | **66.74%** | **The Ceiling.** Subject matter sets the Total Addressable Market. |
| **2** | **Source_Tier_Code** | Context (Identity) | **9.49%** | **The Moat.** Authority outweighs content quality by 6x. |
| **3** | **Topic_Microsoft** | Context (Subject) | **5.39%** | **Utility Limits.** Corporate topics have a lower viral ceiling. |
| **4** | **Topic_Palestine** | Context (Subject) | **2.73%** | **Niche/Global.** Geopolitical topics drive specific engagement. |
| **5** | **hour_sin** | Market/Time | **2.69%** | **Cyclicality.** Publishing during peak traffic matters. |
| **6** | **Title_Complexity** | Content DNA | **2.52%** | **Depth Signal.** Detailed headlines outperform simple ones. |
| **7** | **hour_cos** | Market/Time | **2.07%** | **Cyclicality.** Part of the 24h circadian rhythm vector. |
| **8** | **day_sin** | Market/Time | **1.80%** | **Weekly Cycle.** Weekday vs. Weekend behavior. |
| **9** | **Sentiment_Divergence** | Content DNA | **1.73%** | **Clickbait Check.** Low divergence (Honesty) performs better. |
| **10** | **day_cos** | Market/Time | **1.66%** | **Weekly Cycle.** Weekly seasonality vector. |

### **4. Strategic Insights from the Model**

The "Black Box" revealed four fundamental laws of digital publishing:

*   **1. The "80/20 Rule" of Strategy:**
    *   **Finding:** `Topic` (Subject) and `Source` (Context) account for **~81%** of predictive power. `Sentiment`, `Complexity`, and `Time` account for only **~19%**.
    *   `=>` **Strategy > Tactics.** You cannot "optimize" a bad topic into a viral hit. The decision of *what* to write about dictates the order of magnitude of success.

*   **2. The Distribution Moat:**
    *   **Finding:** `Source_Tier_Code` (#2 Rank) is the strongest non-topic feature.
    *   `=>` **David cannot be Goliath.** Viral velocity is not a meritocracy; it is a function of **Installed Base**. Tier 1 sources have a "Broadcast Privilege" that guarantees a baseline of success.

*   **3. The Complexity Premium:**
    *   **Finding:** `Title_Complexity` (**2.52%**) creates more lift than `Title_Sentiment` (**1.56%**) or `Opportunity_Score` (**1.60%**).
    *   `=>` **Depth is a Feature.** In a clickbait economy, higher cognitive load (longer, detailed headlines) signals utility and correlates with success.

*   **4. The Limit of Prediction (Chaos Factor):**
    *   **Finding:** The best model explains **~25%** of the variance. **75%** remains unexplained.
    *   `=>` **The Ceiling.** We modeled the **Structural Physics** (Who, What, When). The rest is **Stochastic Dynamics** (Luck, Influencer Shares, Dark Social) that no metadata model can predict.

---

## 📂 Project Structure

This repository is organized to separate **Raw Data**, **Engineering Logic**, and **Final Analysis Assets**. **3 master DataDrame .csv files** are too big (~22GB total) to upload directly on Github repository, therefore we provide guidance and download link in *Installation & Getting Started* section below.

```text
Group8_FinalProject/
│
├── Data/
│   ├── prepared/                        # OUTPUT: Analysis-ready datasets
│   │   ├── chapter1_complexity_impact.csv
│   │   ├── chapter1_lifecycle_topic.csv
│   │   ├── chapter1_sentiment_impact.csv
│   │   ├── chapter2_context_hourly.csv
│   │   ├── chapter2_context_opportunity.csv
│   │   ├── chapter2_context_source_tier.csv
│   │   ├── chapter2_context_weekly.csv
│   │   ├── chapter3_golden_quadrant_sample.csv
│   │   ├── chapter3_lifecycle_platform.csv
│   │   ├── experiment_a_results.pkl             # ML: Baseline model artifacts (Raw Data)
│   │   ├── experiment_b_results.pkl             # ML: Strategic model artifacts (Engineered)
│   │   ├── ml_modeling_feature_importance.csv   # ML: Ranked table of predictive drivers
│   │   ├── sec4.4.2_source_map_consolidate.txt  # Log: Source mapping rules
│   │   ├── sec4.5.1_afe_dynamics.txt            # Log: Velocity/Stickiness stats
│   │   ├── sec4.5.2_afe_content_dna.txt         # Log: NLP processing stats
│   │   ├── sec4.5.3_afe_market_ecology.txt      # Log: Opportunity Score stats
│   │   ├── sec5.1.1_df_sample.pkl               # Stratified sample for box plots
│   │   ├── source_mapping.pkl                   # Dictionary for entity resolution
│   │   │
│   │   ├── master_df_consolidated.csv   # [LARGE] The Final Clean 37.5M Row Dataset
│   │   ├── master_df_merged.pkl         # [LARGE] Intermediate state
│   │   └── master_df_temporal.pkl       # [LARGE] Intermediate state
│   │
│   └── [Raw CSV Files]                  # Original source files (Facebook_Economy.csv, etc.)
│
├── figures/                             # Generated Visualizations
│   ├── eda_raw/                         # Initial data exploration plots
│   ├── section*.json                    # Interactive Plotly objects (Preserves zoom/hover capabilities)
│   └── section*.png                     # High-res static exports (For PDF Report & PowerPoint slides)
│
├── A0_Project Kickoff & Initial Direction Setting.ipynb  # Strategic Framework
├── 01_EDA_Raw.ipynb                                     # Diagnostic Analysis
├── 02_Preparation_and_Analysis.ipynb                    # Engineering Pipeline
├── 03_ML_Modeling.ipynb                                 # Machine Learning & Proof
├── dynamics_multicore.py                # [CORE] Parallel Map-Reduce Engine
├── custom_template.py                   # Plotly Styling Configuration
└── all_source_counts.txt                # Log: Raw source frequencies
```
---

## 🔑 Key Files Description

> **⚠️ Important Note to the Reader:**
> The 3 Data Preparation notebooks provides a high-level architectural overview of the project's workflow. The README serves as the roadmap, but the **scientific depth** resides within the individual notebooks.
>
> Each notebook (`0`, `01`, `02`, `03`) is meticulously structured to function as a standalone research document. For every technical step, the notebooks follow a rigorous **Data Science Loop**:
> 1.  **Objective & Methodology:** Defining the *Why* and *How* before coding.
> 2.  **Code Implementation:** Optimized, commented, and robust execution.
> 3.  **Visualization & Output:** Verification of results through immediate feedback.
> 4.  **Analysis:** Deep-dive interpretation of the outputs to justify the next step.

### **1. `0_Project_Kickoff.ipynb` (Strategic Framework)**

*   **Role:** The **Strategic Blueprint** and **Business Case**.
    
    This notebook rejects simple descriptive analysis ("What happened?") in favor of a **Prescriptive Strategy** ("How do we win?"). It establishes the **"Analytics Gap"**—the chasm between the raw data's static nature and the business need for dynamic insights into virality. It defines the **four-dimensional framework** that guides all subsequent engineering.

*   **Structure:**
    ```text
    0. Project Kick-off: Dataset Overview and Analytical Framing
    ├── 0.1. Introduction to the Dataset
    │   (-> Defines the scope: Metadata Anchor + 12 Time-Series Feedback files)
    ├── 0.2. The Core Challenge: Data in an Unusable State
    │   (-> Identifies the 3 barriers: Fragmentation, Structural Rigidity, Integrity Decay)
    ├── 0.3. Key Business Questions: From Analysis to Business Intelligence
    │   (-> Defines the 4 Strategic Chapters for Storytelling)
    │   ├── 0.3.1. Chapter 1: Content DNA (Psycholinguistics: Clickbait, Complexity, Sentiment)
    │   ├── 0.3.2. Chapter 2: Context & Market Ecology (David vs. Goliath, Blue Ocean Strategy)
    │   ├── 0.3.3. Chapter 3: Engagement Dynamics (The Physics of Virality: Speed & Retention)
    │   └── 0.3.4. Chapter 4: Strategy Synthesis (The Golden Quadrant)
    └── 0.4. The Technical Roadmap: Engineering Strategy
        (-> Explicitly maps every Business Question to a Technical Solution)
        ├── 0.4.1. Engineering for Content Analysis (NLP Implementation)
        ├── 0.4.2. Engineering for Engagement Dynamics (Vector Calculus on Time-Series)
        ├── 0.4.3. Engineering for Market Ecology (Categorical Binning)
        └── 0.4.4. Engineering for Market Supply (Rolling Window Algorithims)
    ```

### **2. `01_EDA_Raw.ipynb` (Diagnostic Phase)**

*   **Role:** The **Forensic Diagnostic Audit**.
    
    This notebook acts as a "Data Detective." It does not fix problems; it **proves they exist**. It builds the irrefutable **evidence case** that justifies the complex engineering steps taken later. It systematically diagnoses why standard analysis fails on the raw data (e.g., demonstrating that text columns are analytically silent strings and popularity distributions are mathematically broken by placeholders).

*   **Structure:**
    ```text
    Section 1: Introduction & Setup
    ├── 1.1. General Information (-> Defines the "Diagnostic Investigation" methodology)
    └── 1.2. Library Imports & Global Configuration
    Section 2: Diagnosing the Metadata (News_Final.csv)
    ├── 2.1. Initial Load & Structural Overview (-> Identifies incorrect types: Strings vs Dates)
    ├── 2.2. Popularity Columns: Placeholders and Skewness (-> Visualizes the "Power Law" & -1 Corruption)
    └── 2.3. Categorical Columns: Inconsistency and Granularity
        ├── 2.3.1. Low-Cardinality Categorical ('Topic')
        ├── 2.3.2. High-Cardinality Categorical ('Source') (-> Proves the "Long Tail" noise problem)
        └── 2.3.3. Unstructured Text ('Title', 'Headline') (-> Proves need for NLP extraction)
    Section 3: Diagnosing the Social Feedback Files
    (-> Visualizes the "Wide Format" [144 cols] impossibility for velocity analysis)
    Section 4: Summary of Findings & Handoff
    ├── 4.1. Case for Data Preparation & Engineering
    └── 4.2. Required Preparation & Engineering Actions (-> The formal "Repair Checklist")
    ```

### **3. `02_Preparation_and_Analysis.ipynb` (The Engineering Engine)**

*   **Role:** The **High-Performance Engineering Engine**.
    
    Core implementation file. It executes a **Local Map-Reduce Pipeline** to process **37.5 million rows** without memory failure. It moves beyond cleaning to **Advanced Feature Engineering**, creating the 15 high-value strategic features (Velocity, Stickiness, Opportunity Score) defined in the Kickoff. It concludes by aggregating these millions of data points into concise, insight-rich tables for the final story.

*   **Structure:**
    ```text
    Section 1: Introduction & Setup
    ├── 1.1. General Information (-> Objectives: Integration, Engineering, Aggregation)
    └── 1.2. Library Imports & Global Configuration
    Section 2: Data Integration and Structural Reshaping
    └── 2.1. Combining and Reshaping Feedback Files (-> The "Melt-then-Combine" Memory Strategy)
    Section 3: Merging with Metadata
    └── 3.1. Executing the Inner Merge (-> Creating the 37.5M row Master Backbone)
    Section 4: Data Cleaning & Feature Engineering
    ├── 4.1. Resolving Structural Issues & Data Types
    ├── 4.2. Handling Placeholder Values (-> Nullifying -1s for valid stats)
    ├── 4.3. Temporal Feature Engineering (-> Date Parsing: Hour, Day, Weekend)
    ├── 4.4. Consolidating the 'Source' Column
    │   ├── 4.4.1. Automated Discovery of Source Variations (-> Regex Normalization)
    │   ├── 4.4.2. Applying Standardization and Grouping (-> "Count on Small, Apply to Big" Logic)
    │   └── 4.4.3. Verification of Consolidation
    ├── 4.5. Advanced Feature Engineering (The Factory)
    │   ├── 4.5.1. Dimension I: Dynamics (-> Multiprocessing Vector Calculation: V0 & S)
    │   ├── 4.5.2. Dimension II: Psycholinguistics (-> Parallel NLP: Sentiment & Complexity)
    │   └── 4.5.3. Dimensions III & IV: Market Ecology (-> Rolling Window Opportunity Scores)
    └── 4.6. Final Dataset Verification (-> Polars Scan for Data Density Check)
    Section 5: Final Aggregation for Storytelling
    ├── 5.1. Distribution Analysis & Logarithmic Transformation
    │   ├── 5.1.1. Sampling and Transformation
    │   └── 5.1.2. Impact of Log Transformation (-> Visual Proof: Box Plots)
    ├── 5.2. Chapter 1 Data: Content DNA (Psycholinguistics & Topic)
    │   ├── 5.2.1. Part A: Sentiment & Complexity Analysis (-> Aggregating Content Archetypes)
    │   └── 5.2.2. Part B: Topic Lifecycle Analysis (-> Granular 48h Trajectories)
    ├── 5.3. Chapter 2 Data: Market Context (Ecology & Timing)
    │   ├── 5.3.1. Data Aggregation (-> Multicore processing for Context metrics)
    │   ├── 5.3.2. Ecological Analysis (-> Source Tiering: David vs Goliath)
    │   ├── 5.3.3. Market Analysis (-> Red vs Blue Ocean methodology)
    │   └── 5.3.4. Temporal Analysis (-> Hourly/Weekly Digital Heartbeats)
    └── 5.4. Chapter 3 & 4 Data: Dynamics & Strategy (The Playbook)
        ├── 5.4.1. Part A: Golden Quadrant (-> Scatter Matrix generation)
        └── 5.4.2. Part B: Platform Velocity Lifecycle (-> Initial Velocity Curves)
    Section 6: Final Prepared Datasets (-> Exporting clean CSV assets)
    Section 7: Summary of Findings & Handoff
    ├── 7.1. Execution Summary: From Diagnosis to Engineering
    └── 7.2. Answering the Strategic Questions (-> Confirmation of Analytical Success)
    ```

### **4. `03_ML_Modeling.ipynb` (The Proof Phase)**

*   **Role:** The **Scientific Validation & A/B Experiment**.

    This notebook serves as the "Scientific Courtroom." It executes a rigorous **A/B Test** to quantify the exact value of the Feature Engineering performed in Phase 2. Instead of optimizing hyperparameters, it compares two distinct datasets using the same algorithm: **Model A (The Lazy Baseline)** trained on raw metadata, versus **Model B (The Strategic Solution)** trained on engineered features. It provides the mathematical proof that data preparation drives performance, not just algorithm choice.

*   **Structure:**
    ```text
    Section 1: The Mathematical Challenge (The Power Law Trap)
    ├── 1.1. Objective & Target Definition (-> Moving from Classification to Regression)
    └── 1.2 - 1.4. The "Unmodelable" Reality (-> Visualizing why raw distribution breaks linear models)
    Section 2: Experiment A - The "Lazy Analyst" (The Baseline)
    ├── 2.1. Experimental Design (-> Simulating the "Garbage In" approach)
    └── 2.3. Analysis of Failure (-> Proving that raw metadata has near-zero predictive signal)
    Section 3: Experiment B - The "Strategic Data Scientist" (The Solution)
    ├── 3.1. Methodology: The "Engineered" Pipeline (-> Applying Log-Normal & Context features)
    └── 3.3. Analysis of Victory (-> Quantifying the "Signal-to-Noise" revolution)
    Section 4: Comparative Evaluation & Forensic Analysis
    ├── 4.1. Visualizing the "Lift" (-> Scatter Plot: Flatline vs. Diagonal Trend)
    ├── 4.2. Visualizing Behavior: Prediction Density (-> KDE Plot: "Delusional Average" vs. Reality)
    ├── 4.3. Feature Importance (-> Ranking the "Drivers of Virality": Subject vs. Context)
    └── 4.4. Strategic Insights (-> The "Viral Architecture" business playbook)
    Section 5: Final Verdict
    ├── 5.1. Executive Summary: The A/B Test Results (-> The Comparative Matrix)
    └── 5.2. Core Lesson: Quality Over Quantity (-> Closing the "Data-Centric AI" case)
    ```

### **5. `dynamics_multicore.py` (The Parallel Engine)**

*   **Role:** **High-Performance Compute Module**.
    
    Standard Python execution is single-threaded (bound by the GIL), making complex feature engineering on **37.5 million rows** computationally prohibitive. This custom module bypasses that limitation by implementing a **Parallel Map-Reduce Architecture**, enabling the processing pipeline to run in **6-8 minutes** (a ~5.5x speedup over sequential processing).

*   **Key Functions & Architectural Logic:**
    *   **`get_file_chunks` (Byte-Aligned Sharding):**
        *   Instead of reading rows sequentially, this function calculates byte offsets to split the massive CSV into logical shards (e.g., 30 chunks).
        *   **Crucial Optimization:** It aligns splits to the nearest newline character (`\n`), allowing workers to `seek()` directly to their assigned block without parsing preceding data.
    *   **`read_csv_chunk` (Random Access I/O):**
        *   A specialized reader that executes the "Map" phase. It jumps to specific byte coordinates to read strictly defined data segments into memory.
    *   **The Two-Pass Algorithms:**
        *   **Pass 1 (Map/Extraction):** Worker functions like `process_phase1_range` scan shards to isolate global metrics (e.g., finding the `TimeSlice=1` row for Velocity or the `Max(Popularity)` for Final Score). These are aggregated in the main thread into a lightweight Lookup Table.
        *   **Pass 2 (Reduce/Broadcast):** Worker functions like `process_phase2_merge` read the shards again, performing a "Left Join" with the Lookup Table to calculate vector metrics (Stickiness) row-by-row.
    *   **`process_phase2_merge` (Sharded Writing):**
        *   To avoid I/O bottlenecks, workers do not write to a single file. Instead, they output independent temporary shards (`part_001.csv`, etc.), which are binary-concatenated at the end of the pipeline.

*   **Strategic Note:** This file represents the project's shift from "Scripting" to **"Software Engineering."** It demonstrates the ability to handle Big Data constraints on local hardware through efficient resource management and parallel algorithmic design.

---

## ⚙️ Installation & Getting Started

### 1. Environment Setup  
```bash
git clone https://github.com/quanninja1304/DSEB65A_Group8_Final-Project
pip install -r requirements.txt
```

---

### 2. Data Setup & Execution Modes

Due to GitHub's file size limits (100MB), the 3 master datasets (`master_df_consolidated.csv`, `master_df_merged.csv`, `master_df_temporal.csv`) required for this project (~22GB total, .zip files ~6GB) cannot be hosted directly in the repository.

You have **two options** to run this project, depending on whether you want to verify the results (Fast) or replicate the engineering process (Slow).

#### 🚀 Option 1: Fast Track (Recommended)

**Use Case:** You want to run the notebooks immediately to see results, visualizations, and models without waiting for heavy data processing.

1.  **Clone/Download** this repository to your local machine.
2.  **Download the Large Data Bundle (.zip)** from Google Drive:  
    👉 [Google Drive Link](https://drive.google.com/drive/folders/1TtXHIht7n5iTJx2qx8-hjjDHJxwsoPg4?usp=drive_link).
3.  **Extract** the contents of the zip file into the `Data/prepared/` folder inside your project directory.
4.  **Verify Structure:** Your folder must look like this:
    ```text
    Group8_FinalProject/
    ├── Data/
    │   ├── prepared/
    │   │   ├── master_df_consolidated.csv  <-- From Zip
    │   │   ├── master_df_merged.pkl        <-- From Zip
    │   │   ├── master_df_temporal.pkl      <-- From Zip
    │   │   ├── [other .csv/.txt files included in repo...]
    ```
5.  **Run:**
    *   `01_EDA_Raw.ipynb`: Runs instantly.
    *   `02_Preparation_and_Analysis.ipynb`: Takes **6-15 minutes** (loads checkpoints instead of processing).
    *   `03_ML_Modeling.ipynb`: Runs instantly (loads saved model results).


#### 🛠️ Option 2: Build from Scratch (Full Replication)

**Use Case:** You want to audit the code logic and rebuild the datasets from the raw source files.

1.  **Clone/Download** this repository.
2.  **Ensure** the `Data/` folder contains `News_Final.csv` and the 12 social feedback CSVs (included in this repo).
3.  **Do NOT** download the files from Google Drive. Ensure the `Data/prepared/` folder does **not** contain the 3 master files listed above.
4.  **Run:**
    *   Execute `02_Preparation_and_Analysis.ipynb`.
    *   **Consequence:** The code will detect missing master files and trigger the **Full Engineering Engine**.
    *   **Time Required:** Approximately **~60 minutes** (depending on CPU/RAM) to process 37.5 million rows, perform Map-Reduce engineering, and save the results.

---

### 3. Project Workflow

Once your data is set up (via Option 1 or 2 above), execute the notebooks in this specific order to replicate our full analysis path:

#### **Step 1: Strategic Foundation**
*   **Run:** `A0_Project Kickoff & Initial Direction Setting.ipynb`
*   **Purpose:** Reviews the project charter, hypotheses, and analytical framework.

#### **Step 2: The Diagnosis**
*   **Run:** `01_EDA_Raw.ipynb`
*   **Purpose:** Performs the forensic audit of the raw data. It visualizes the "Power Law" distribution and proves why standard analysis fails without engineering.

#### **Step 3: The Core Engine (Engineering & Storytelling)**
*   **Run:** `02_Preparation_and_Analysis.ipynb`
*   **Purpose:**
    *   **Engineering:** Triggers the Multicore Map-Reduce engine to calculate Velocity, Stickiness, and Sentiment.
    *   **Storytelling:** Generates the interactive Plotly visualizations for Chapters 1-4.
    *   *Note: If you chose Option 1 (Fast Track), this runs in minutes. If Option 2, it runs the full build.*

#### **Step 4: The Scientific Proof (Machine Learning)**
*   **Run:** `03_ML_Modeling.ipynb`
*   **Purpose:** Conducts the rigorous **A/B Experiment**. It trains two models (Baseline vs. Strategic) to mathematically prove that our Feature Engineering drove a **480% performance improvement**.

--- 

## 📜 License

This project is licensed under the **MIT License** — see the `LICENSE` file for details.
