# Advanced System & ML Workflows

This document outlines the detailed architectural workflows powering the AI Financial Advisor platform. Traditional diagrams often miss cross-dependencies (e.g. how the Anomaly detector requires Profile Savings to calculate risk ratios). These updated workflows map exactly how the backend actually breathes.

---

## 1. Complete System Architecture & Data Flow 

This diagram captures the runtime interaction between the Frontend Dashboard, Database, Machine Learning Inference Engines, and External APIs (like Yahoo Finance and Google Gemini).

```mermaid
graph TD
    A[User Sign in / Sign up] --> B[User inputs financial profile<br>income, expenses, savings, etc.]
    B --> C[User Adds Transactions<br>Income / Expense Records]
    C --> D[Financial Data Stored in Database]
    D --> E[Data Preprocessing + Feature Extraction]
    E --> F[Load Trained Models<br>Risk, Anomaly Detection, Spending Forecasting]
    F --> G[Financial Analysis Engine<br>Risk + Budget Optimization + Forecast + Anomaly Detection]

    G --> Dash
    G --> AIMod
    G --> StockMod

    subgraph Dashboard Module
        Dash[Show Insights<br>Risk score, charts, forecasts, alerts]
    end

    subgraph AI Assistant Module
        direction TB
        AIMod[User Query Input] --> Intent[Intent Detection / Routing]
        Intent --> Fin[Financial Query --> LLM Response --> Chat Response]
        Intent --> StQ[Stock Query --> Redirect to Stock Analysis]
    end

    subgraph Stock Analysis Module
        direction TB
        StockMod[Fetch Stock Data - yFinance API] --> Anal[Stock Analysis - Gemini]
        Anal --> Disp[Display Stock Insights]
    end
    
    %% Semantic Styling
    classDef main fill:#f9f9f9,stroke:#333,stroke-width:2px;
    class A,B,C,D,E,F,G main;
```

---

## 2. Machine Learning Training Workflow

This defines how the local Python environment extracts base financial data, executes feature engineering (critical step), retrains the statistical algorithms, and outputs the `.pkl` schema binaries used by the system above.

```mermaid
graph TD
    %% Base Data
    RawData[(Raw Financial CSVs)]
    
    %% Data Processing
    RawData -->|Pandas Extraction| Preprocessor[Data Preprocessing Module]
    
    %% Feature Engineering Splits
    Preprocessor -->|User Snapshots| ProfileEngineering[Profile Feature Engineering]
    ProfileEngineering -->|Create| ExpenseRatio[expense_ratio]
    ProfileEngineering -->|Create| DebtRatio[debt_ratio]
    ProfileEngineering -->|Calc| Utilization[budget_utilization]

    Preprocessor -->|Time-Series Extractions| TimeEngineering[Temporal Data Engineering]
    TimeEngineering -->|Impact Ratio| Impact[transaction_impact = amount / savings]
    TimeEngineering -->|Daily Aggregate| Daily[daily_spending_sums]

    %% Model Training Vectors
    ExpenseRatio --> ReLUModel[Deep Learning Behavioral Model - ReLU]
    DebtRatio --> ReLUModel
    Utilization --> ReLUModel
    
    Impact --> IsoForest[Isolation Forest Detector]
    
    Daily --> Prophet[Prophet Temporal Forecaster]

    %% Weights Compilation
    ReLUModel -->|PyTorch Compilation| PKL1[[risk_model.pkl]]
    IsoForest -->|Scikit-learn Compilation| PKL2[[isolation_forest.pkl]]
    Prophet -->|Stan Compilation| PKL3[[spending_forecaster.pkl]]

    %% Serialization 
    PKL1 --> |Load on Uvicorn Boot| BackendServer([FastAPI Backend /app/engine/])
    PKL2 --> |Load on Uvicorn Boot| BackendServer
    PKL3 --> |Load on Uvicorn Boot| BackendServer

    %% Semantic Styling
    classDef data fill:#f2cdac,stroke:#333,stroke-width:2px;
    classDef process fill:#fff9c4,stroke:#333,stroke-width:2px;
    classDef algo fill:#c8e6c9,stroke:#333,stroke-width:2px;
    classDef pkl fill:#ffd6e5,stroke:#333,stroke-width:2px;
    classDef server fill:#b8c0ff,stroke:#333,stroke-width:2px;

    class RawData data;
    class Preprocessor,ProfileEngineering,TimeEngineering process;
    class ReLUModel,IsoForest,Prophet algo;
    class PKL1,PKL2,PKL3 pkl;
    class BackendServer server;
```
