# Architecture Diagram

## New Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER / LLM                              │
│  "Load my sales data from PostgreSQL and forecast next week"   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MCP PROTOCOL LAYER                           │
│  server.py - Handles MCP communication                          │
│  • load_data_source                                             │
│  • fit_predict_with_data                                        │
│  • list_data_sources                                            │
│  • list_data_handles                                            │
│  • release_data_handle                                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      TOOLS LAYER                                │
│  data_tools.py - MCP tool implementations                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EXECUTOR LAYER                               │
│  executor.py - Manages estimators and data                      │
│  • _data_handles = {}  (NEW)                                    │
│  • load_data_source()  (NEW)                                    │
│  • fit_predict_with_data()  (NEW)                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  DATA SOURCE LAYER (NEW)                        │
│  registry.py - Adapter registry                                 │
│  ┌───────────────┬───────────────┬───────────────┐             │
│  │ PandasAdapter │  SQLAdapter   │  FileAdapter  │             │
│  │               │               │               │             │
│  │ • load()      │ • load()      │ • load()      │             │
│  │ • validate()  │ • validate()  │ • validate()  │             │
│  │ • to_sktime() │ • to_sktime() │ • to_sktime() │             │
│  └───────┬───────┴───────┬───────┴───────┬───────┘             │
│          │               │               │                     │
└──────────┼───────────────┼───────────────┼─────────────────────┘
           │               │               │
           ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ DataFrame│   │PostgreSQL│   │   CSV    │
    │   Dict   │   │  MySQL   │   │  Excel   │
    │          │   │  SQLite  │   │ Parquet  │
    └──────────┘   └──────────┘   └──────────┘
```

## Component Interaction

```
1. User Request
   └─> "Load data from CSV and forecast"

2. MCP Server
   └─> Routes to load_data_source tool

3. Data Tools
   └─> Calls executor.load_data_source(config)

4. Executor
   └─> Creates adapter from registry
   └─> Calls adapter.load()
   └─> Calls adapter.validate()
   └─> Calls adapter.to_sktime_format()
   └─> Stores in _data_handles
   └─> Returns data_handle

5. User Request
   └─> "Fit ARIMA and predict"

6. MCP Server
   └─> Routes to fit_predict_with_data tool

7. Executor
   └─> Retrieves data from _data_handles
   └─> Fits estimator
   └─> Generates predictions
   └─> Returns results
```

## Data Adapter Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                   DataSourceAdapter (ABC)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ + load() -> DataFrame                                    │   │
│  │ + validate(data) -> (bool, report)                       │   │
│  │ + to_sktime_format(data) -> (y, X)                       │   │
│  │ + get_metadata() -> dict                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Pandas    │  │     SQL     │  │    File     │
│   Adapter   │  │   Adapter   │  │   Adapter   │
│             │  │             │  │             │
│ • Dict      │  │ • Postgres  │  │ • CSV       │
│ • DataFrame │  │ • MySQL     │  │ • Excel     │
│ • Auto-     │  │ • SQLite    │  │ • Parquet   │
│   detect    │  │ • MSSQL     │  │ • TSV       │
│   time col  │  │ • Query     │  │ • Auto-     │
│             │  │   builder   │  │   detect    │
└─────────────┘  └─────────────┘  └─────────────┘
```

## Data Validation Flow

```
Data Source
    │
    ▼
adapter.load()
    │
    ├─> Set time index
    ├─> Sort by time
    ├─> Infer/set frequency
    │
    ▼
adapter.validate()
    │
    ├─> Check DatetimeIndex ✓
    ├─> Check for duplicates ✓
    ├─> Check missing values ⚠
    ├─> Check monotonic ⚠
    ├─> Check frequency ⚠
    ├─> Check data size ⚠
    │
    ▼
Validation Report
    {
      "valid": true/false,
      "errors": [...],      # Critical issues
      "warnings": [...]     # Non-critical issues
    }
    │
    ▼
adapter.to_sktime_format()
    │
    ├─> Extract target (y)
    ├─> Extract exogenous (X)
    │
    ▼
(y, X) ready for sktime
```

## Handle Management

```
┌─────────────────────────────────────────────────────────────────┐
│                    Executor._data_handles                       │
│  {                                                              │
│    "data_abc123": {                                             │
│      "y": Series(...),                                          │
│      "X": DataFrame(...),                                       │
│      "metadata": {...},                                         │
│      "validation": {...},                                       │
│      "config": {...}                                            │
│    },                                                           │
│    "data_xyz789": {...}                                         │
│  }                                                              │
└─────────────────────────────────────────────────────────────────┘
         │                                    │
         │ fit_predict_with_data()            │ release_data_handle()
         │                                    │
         ▼                                    ▼
    Retrieve data                        Delete handle
    Fit estimator                        Free memory
    Generate predictions
```

## Complete Workflow Example

```
1. Load Data
   ┌──────────────────────────────────────────────────────────┐
   │ load_data_source({                                       │
   │   "type": "sql",                                         │
   │   "connection_string": "postgresql://...",               │
   │   "query": "SELECT * FROM sales",                        │
   │   "time_column": "date",                                 │
   │   "target_column": "revenue"                             │
   │ })                                                       │
   └────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
   ┌──────────────────────────────────────────────────────────┐
   │ Returns:                                                 │
   │ {                                                        │
   │   "success": true,                                       │
   │   "data_handle": "data_abc123",                          │
   │   "metadata": {                                          │
   │     "rows": 1000,                                        │
   │     "columns": ["revenue", "temperature"],               │
   │     "frequency": "D",                                    │
   │     "start_date": "2020-01-01",                          │
   │     "end_date": "2022-09-27"                             │
   │   },                                                     │
   │   "validation": {                                        │
   │     "valid": true,                                       │
   │     "warnings": []                                       │
   │   }                                                      │
   │ }                                                        │
   └────────────────────┬─────────────────────────────────────┘
                        │
2. Instantiate Model   │
   ┌──────────────────────────────────────────────────────────┐
   │ instantiate_estimator("ARIMA", {"order": [1,1,1]})       │
   └────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
   ┌──────────────────────────────────────────────────────────┐
   │ Returns: {"handle": "est_xyz789"}                        │
   └────────────────────┬─────────────────────────────────────┘
                        │
3. Fit & Predict       │
   ┌──────────────────────────────────────────────────────────┐
   │ fit_predict_with_data(                                   │
   │   estimator_handle="est_xyz789",                         │
   │   data_handle="data_abc123",                             │
   │   horizon=7                                              │
   │ )                                                        │
   └────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
   ┌──────────────────────────────────────────────────────────┐
   │ Returns:                                                 │
   │ {                                                        │
   │   "success": true,                                       │
   │   "predictions": {                                       │
   │     "2022-09-28": 1250.5,                                │
   │     "2022-09-29": 1255.2,                                │
   │     ...                                                  │
   │   },                                                     │
   │   "horizon": 7                                           │
   │ }                                                        │
   └──────────────────────────────────────────────────────────┘
```
