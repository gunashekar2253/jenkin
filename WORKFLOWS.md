# Advanced System & ML Workflows

This document outlines the detailed architectural workflows powering the AI Financial Advisor platform. Traditional diagrams often miss cross-dependencies (e.g. how the Anomaly detector requires Profile Savings to calculate risk ratios). These updated workflows map exactly how the backend actually breathes.

---

## 1. Complete System Architecture & Data Flow 

This diagram captures the runtime interaction between the Frontend Dashboard, Database, Machine Learning Inference Engines, and External APIs (like Yahoo Finance and Google Gemini).

```mermaid
graph TD
    %% Core User Actions
    Client((User / Frontend))
    
    %% Auth & Profile
    Client -->|Register / Login| AuthRouter[Auth Router / JWT]
    AuthRouter -->|Store| DB[(PostgreSQL / SQLite Database)]
    Client -->|Update Income/Savings| ProfileRouter[Profile Settings Router]
    ProfileRouter -->|Save Config| DB
    
    %% Transactions
    Client -->|Log Expense / Income| TxnRouter[Transactions Router]
    TxnRouter -->|Append| DB
    
    %% Financial Analysis & Dashboard
    Client -->|Request Dashboard Data| DashboardRouter[Analysis Router]
    DashboardRouter -->|Fetch History & Profile| DB
    
    %% Core ML Inference Layer
    DashboardRouter -->|Trigger Inference| MLEngine{AI Financial Engine}
    
    MLEngine -->|Profile Mappings| RiskModel(XGBoost Risk Predictor)
    MLEngine -->|Transaction Impacts| AnomalyModel(Isolation Forest Anomaly Detector)
    MLEngine -->|Time-Series Dates| ProphetModel(Prophet Spending Forecaster)
    
    RiskModel -->|Scores & Heuristics| DashboardRouter
    AnomalyModel -->|Severity Ratings| DashboardRouter
    ProphetModel -->|Future Budget| DashboardRouter
    
    DashboardRouter -->|Render Charts & Alerts| Client

    %% Stock Analysis Module
    Client -->|Search Ticker e.g. TCS| StockRouter[Stocks Router]
    StockRouter -->|1. Clean Query & Rules| TickerLogic(_resolve_ticker_symbol)
    
    TickerLogic -->|2. Check Cache| Cache[In-Memory TTLCache]
    Cache -->|Cache Miss| YF[Yahoo Finance API]
    YF -->|3. Fallback .NS / .BO| YF
    YF -->|Raw JSON Data| StockRouter
    StockRouter -->|Save to Cache| Cache
    
    StockRouter -->|4. Feed Payload| LLMAgent[Stock Agent]
    LLMAgent -->|Rest HTTP POST| Gemini[Google Gemini 1.5 Flash]
    Gemini -->|Generative Summary| LLMAgent
    LLMAgent -->|Return Stock Page| Client
    
    %% Semantic Styling
    classDef database fill:#f2cdac,stroke:#333,stroke-width:2px;
    classDef engine fill:#b8c0ff,stroke:#333,stroke-width:2px;
    classDef model fill:#c8e6c9,stroke:#333,stroke-width:2px;
    classDef external fill:#ffd6e5,stroke:#333,stroke-width:2px;
    classDef router fill:#ffe0b2,stroke:#333,stroke-width:2px;
    
    class DB database;
    class MLEngine engine;
    class RiskModel,AnomalyModel,ProphetModel model;
    class Gemini,YF external;
    class AuthRouter,ProfileRouter,TxnRouter,DashboardRouter,StockRouter router;

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
    ExpenseRatio --> XGBoost[XGBoost Behavior Model]
    DebtRatio --> XGBoost
    Utilization --> XGBoost
    
    Impact --> IsoForest[Isolation Forest Detector]
    
    Daily --> Prophet[Prophet Temporal Forecaster]

    %% Weights Compilation
    XGBoost -->|Scikit-learn Compilation| PKL1[[risk_model.pkl]]
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
    class XGBoost,IsoForest,Prophet algo;
    class PKL1,PKL2,PKL3 pkl;
    class BackendServer server;
```
