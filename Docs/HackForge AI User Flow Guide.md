# **HackForge AI: Open-Source User Flow & Micro-Interactions Guide**

## **1\. Design Philosophy (Open Source Edition)**

This user flow is designed for a self-hosted or open-source community edition of HackForge AI.

* **Hardware-Light:** Focuses exclusively on tabular data (numeric, categorical, boolean, datetime, text), causal graphs, and failure injection. Heavy multi-modal features (GANs, VAEs) are excluded.  
* **Cost-Free:** Removes all token buckets, rate limiting, billing, and credit estimations.  
* **Highly Interactive:** Emphasizes instant feedback, seamless transitions, and transparent execution to help developers and data scientists understand the deterministic generation process.

## **2\. Phase 1: Discovery & Onboarding**

### **2.1 Landing Page**

* **Visuals:** A clean, developer-focused aesthetic (dark mode by default).  
* **Content:** "Simulate Realities, Not Just Rows. The open-source deterministic synthetic data engine."  
* **Primary CTA:** "Start Generating (Free/Local)"  
* **Secondary CTAs:** "Browse Templates," "GitHub Repo," "Documentation."  
* **Micro-Interactions:**  
  * *Hover on CTA:* Slight glow effect and scale up (1.05x).  
  * *Scroll:* A lightweight, interactive terminal window auto-types a sample HackForge YAML configuration, demonstrating the schema DSL.

### **2.2 Authentication (Optional for Local, Required for Cloud-hosted OSS)**

* **Flow:** Standard OAuth (GitHub, Google) or basic Email/Password.  
* **Micro-Interactions:**  
  * *Input Focus:* Highlight border with the primary brand color.  
  * *Submit:* Button transitions to a loading spinner. Success triggers a smooth slide-up transition to the Dashboard.

## **3\. Phase 2: The Dashboard (Home Base)**

### **3.1 Overview**

The central hub for managing experiments and accessing hackathon templates.

* **Layout:** Left sidebar for navigation (My Datasets, Templates, Settings). Main area split into "Recent Activity" and "Your Datasets".  
* **Dataset Cards:** Display Name, Creation Date, Status (Draft, Running, Completed, Failed), Row Count, and Feature Count.  
* **Empty State:** If no datasets exist, display an illustration of an empty laboratory with two large pulse-animated buttons: "Create Blank Dataset" and "Use a Template".  
* **Micro-Interactions:**  
  * *Status Badges:* 'Running' features a pulsing blue dot; 'Completed' a solid green check; 'Failed' a red warning triangle.  
  * *Card Hover:* Elevates slightly with a shadow drop; reveals quick-action icons (Duplicate, Delete, Export).

### **3.2 Hackathon Templates Tab**

* **Content:** Cards for predefined schemas (e.g., "Financial Fraud", "Healthcare Readmission", "E-Commerce Churn").  
* **Micro-Interactions:** Clicking a template triggers a slide-over panel previewing the schema nodes and causal graph before the user confirms "Use Template".

## **4\. Phase 3: Dataset Initialization**

### **4.1 Setup Modal**

Triggered by clicking "Create Dataset".

* **Fields:**  
  * **Name** (Required): Auto-focuses on open.  
  * **Description** (Optional).  
  * **Row Count** (Required): Slider \+ Input field (e.g., 1,000 to 1,000,000).  
  * **Difficulty Target** (Dropdown): Beginner, Intermediate, Advanced, Kaggle.  
* **Micro-Interactions:**  
  * *Row Count Slider:* As the user drags the slider, a real-time "Estimated File Size" (e.g., \~15MB) updates dynamically based on an assumed baseline.  
  * *Submit:* Transitions smoothly into the main Canvas workspace.

## **5\. Phase 4: The Canvas (Core Workspace)**

This is the primary interface. It contains a Top Toolbar, a Main Working Area (Table/Graph toggle), and a Right Inspector Panel.

### **5.1 Top Toolbar**

* **Components:** View Toggle (Table / Graph), Seed Input (optional override, masked if requested), "Validate" button, "Estimate Run" button, "Generate" (Primary CTA).  
* **Micro-Interactions:** \* *Seed Input:* Hovering shows a tooltip: "Leave blank for a randomly generated deterministic seed."

### **5.2 Table View (Schema Builder)**

Behaves like a smart spreadsheet, but defines *schema*, not data.

* **Layout:** Empty rows below column headers.  
* **Micro-Interactions:**  
  * *Add Column:* Clicking the '+' icon at the end of the headers instantly adds a new column "Feature\_X".  
  * *Rename:* Double-clicking a column header changes it to a text input field. Pressing Enter saves it.  
  * *Reorder:* Click and hold a column header to drag and drop it horizontally.  
  * *Active State:* Clicking a column highlights it and opens its properties in the Right Inspector Panel.

### **5.3 Right Inspector Panel (Contextual)**

Changes based on the selected column type.

* **Numeric Type Selected:** Shows Dropdowns for Distribution (Normal, Poisson, Pareto), Inputs for Mean, Variance, Min/Max.  
* **Categorical Type Selected:** Shows a dynamic list builder for Categories and an interactive pie-chart for Class Weights/Imbalance.  
* **Relationships Section:** A UI to define causal parents.  
* **Micro-Interactions:**  
  * *Validation feedback:* Typing a negative variance immediately turns the input border red and displays a micro-tooltip: "Variance must be positive."  
  * *Adding Parents:* Clicking "Add Dependency" opens a dropdown of other columns. Selecting one draws a subtle, temporary arrow from that column to the current one if the Table View is visible.

### **5.4 Graph View (Causal Visualizer)**

* **Layout:** A node-based 2D canvas (using a library like React Flow). Columns are nodes; relationships are directed edges.  
* **Micro-Interactions:**  
  * *Drag & Drop:* Nodes can be dragged to organize the visual layout.  
*   
  * *Edge Creation:* Dragging from a "handle" on Node A to Node B creates a causal link.  
  * *Cycle Detection:* If the user connects Node B back to Node A, the edge instantly turns red, and a toast notification pops up: "Invalid: Causal loops (cycles) are not permitted."

## **6\. Phase 5: Pre-Flight & Generation**

### **6.1 Validation & Estimation**

Before generation, the user clicks "Estimate Run" or "Generate" (which triggers auto-validation).

* **Validation Panel:** Slides down from the top toolbar. Checks DAG acyclicity, parameter validity, and missing targets.  
* **Estimation Modal (OSS Adapted):** \* *Shows:* Estimated Runtime (e.g., "\~45 seconds"), Estimated RAM usage (e.g., "1.2 GB"), Expected File Size.  
  * *Removed:* Cost estimates, credit deductions, and GPU allocation (assumes CPU-bound local/open-source execution).  
* **Micro-Interactions:** A smooth progress bar loads the estimates. The "Generate Dataset" button unlocks and pulses.

### **6.2 Generation Tracker (Real-Time)**

The Canvas fades back, and a centered "Event Tracker" terminal appears.

* **Stages Listed:** Intake → DAG Sorting → Base Generation → Equation Simulation → KS-Test Validation → Packaging.  
* **Micro-Interactions:**  
  * *Live Streaming:* Uses WebSockets to check off stages one by one. The current stage has a spinning icon.  
  * *Console Output:* A small, expandable terminal window shows raw backend logs (e.g., \[INFO\] Generated 50,000 rows for feature 'Age').  
  * *Completion:* The screen flashes a subtle success color, and a "View Results" button appears.

## **7\. Phase 6: Results & Evaluation**

### **7.1 Clean Dataset View**

* **Layout:** Split into Tabs: Data Preview, Schema Summary, Distributions, Correlation Matrix, Evaluation Report.  
* **Data Preview Tab:** A paginated data grid showing the first 100 rows.  
* **Evaluation Report Tab:** Shows Statistical Compliance Score (e.g., 98%). Lists features with Target vs. Actual metrics.  
* **Micro-Interactions:**  
  * *Hover on Correlation Matrix:* Hovering over a cell (e.g., Age vs. Income) shows a tooltip with the exact Pearson correlation coefficient.

### **7.2 The Failure Injection Fork**

A prominent call-out box at the bottom of the Evaluation Report: *"Test your ML model's robustness. Do you want to inject failures?"*

* **Buttons:** "Keep Clean Dataset Only" (Secondary) | "Create Failure-Injected Variant" (Primary).

### **7.3 Failure Injection Configurator**

If the user chooses to inject failures:

* **Layout:** Split screen. Clean dataset stats on the left, injection tools on the right.  
* **Tools Available:** Accordions for Missing Data (MCAR/MAR), Adversarial Noise, Concept Drift.  
* **Micro-Interactions:**  
  * *Toggles:* Activating "Missing Data" expands a slider to choose the % of missingness and a multi-select dropdown for target columns.  
  * *Live Diff Preview:* Adjusting a slider updates a "Expected Impact" visual on the left (e.g., showing a bar chart of data retention dropping).  
  * *Execution:* Clicking "Inject" brings up a mini-version of the Event Tracker.

### **7.4 Split Comparison View**

Once injected, the UI defaults to a Comparison Tab.

* **Layout:** Side-by-side or overlay views of Clean vs. Injected.  
* **Content:** Highlights exact changes (e.g., "Income column now contains 15% Nulls", "Age variance shifted by \+2.4").

## **8\. Phase 7: Export & Lifecycle**

### **8.1 Export Flow**

Available from the Results page or Dashboard.

* **Modal Options:**  
  * *Dataset Version:* Clean, Injected, or Both.  
  * *Format:* CSV, JSON, Parquet.  
  * *Include Metadata:* Checkbox to include config.json and report.pdf.  
* **Micro-Interactions:** Clicking "Download" triggers a local file generation process with a progress circle on the button.

### **8.2 Dashboard Post-Actions**

Back on the dashboard, clicking the '...' menu on a completed dataset reveals:

* **"Regenerate (Same Seed)":** Instantly queues a job with identical parameters.  
* **"Edit as New Version":** Duplicates the configuration into a new Draft experiment and opens the Canvas. (Crucial for respecting the immutable configuration backend rule).