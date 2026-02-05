# Data Source Support

sktime-mcp now supports loading data from multiple sources beyond the built-in demo datasets!

## Supported Data Sources

1. **Pandas DataFrames** - In-memory data
2. **SQL Databases** - PostgreSQL, MySQL, SQLite, MSSQL
3. **Files** - CSV, TSV, Excel, Parquet

## Quick Start

### 1. Pandas DataFrame

```python
from sktime_mcp.runtime.executor import get_executor
import pandas as pd

# Create your data
df = pd.DataFrame({
    'date': pd.date_range(start='2020-01-01', periods=100, freq='D'),
    'sales': [100 + i for i in range(100)],
})

# Load data
executor = get_executor()
result = executor.load_data_source({
    "type": "pandas",
    "data": df,  # Can also be a dict
    "time_column": "date",
    "target_column": "sales",
})

# Get data handle
data_handle = result['data_handle']

# Instantiate model and predict
est_result = executor.instantiate("NaiveForecaster", {"strategy": "last"})
predictions = executor.fit_predict_with_data(
    estimator_handle=est_result['handle'],
    data_handle=data_handle,
    horizon=7,
)
```

### 2. CSV File

```python
result = executor.load_data_source({
    "type": "file",
    "path": "/path/to/data.csv",
    "time_column": "date",
    "target_column": "sales",
    "exog_columns": ["temperature", "promotion"],  # Optional
})
```

### 3. SQL Database

```python
# SQLite
result = executor.load_data_source({
    "type": "sql",
    "connection_string": "sqlite:///path/to/database.db",
    "query": "SELECT date, sales FROM sales_table",
    "time_column": "date",
    "target_column": "sales",
})

# PostgreSQL
result = executor.load_data_source({
    "type": "sql",
    "connection_string": "postgresql://user:pass@host:5432/db",
    "query": "SELECT * FROM sales WHERE date >= '2020-01-01'",
    "time_column": "date",
    "target_column": "sales",
})
```

## MCP Tools

### `load_data_source`

Load data from any supported source.

**Arguments:**
- `config` (dict): Data source configuration

**Returns:**
- `success` (bool): Whether loading succeeded
- `data_handle` (str): Handle to reference the loaded data
- `metadata` (dict): Information about the data (rows, columns, frequency, etc.)
- `validation` (dict): Data quality validation results

**Example:**
```json
{
  "config": {
    "type": "pandas",
    "data": {"date": [...], "value": [...]},
    "time_column": "date",
    "target_column": "value"
  }
}
```

### `fit_predict_with_data`

Fit a model and generate predictions using custom data.

**Arguments:**
- `estimator_handle` (str): Handle from `instantiate_estimator`
- `data_handle` (str): Handle from `load_data_source`
- `horizon` (int): Forecast horizon (default: 12)

**Example:**
```json
{
  "estimator_handle": "est_abc123",
  "data_handle": "data_xyz789",
  "horizon": 7
}
```

### `list_data_sources`

List all available data source types.

**Returns:**
- `sources` (list): Available source types
- `descriptions` (dict): Description for each source

### `list_data_handles`

List all currently loaded data handles.

**Returns:**
- `count` (int): Number of loaded data handles
- `handles` (list): List of data handle information

### `release_data_handle`

Release a data handle and free memory.

**Arguments:**
- `data_handle` (str): Handle to release

## Configuration Options

### Pandas Adapter

```python
{
    "type": "pandas",
    "data": df_or_dict,  # DataFrame or dict
    "time_column": "date",  # Optional, will try to detect
    "target_column": "sales",  # Optional, defaults to first column
    "exog_columns": ["temp", "promo"],  # Optional
    "frequency": "D"  # Optional, will try to infer
}
```

### SQL Adapter

```python
{
    "type": "sql",
    
    # Option 1: Connection string
    "connection_string": "postgresql://user:pass@host:5432/db",
    
    # Option 2: Individual components
    "dialect": "postgresql",  # postgresql, mysql, sqlite, mssql
    "host": "localhost",
    "port": 5432,
    "database": "mydb",
    "username": "user",
    "password": "pass",
    
    # Query
    "query": "SELECT * FROM sales",  # Direct SQL query
    # OR
    "table": "sales",  # Table name
    "filters": {"region": "North"},  # Optional filters
    
    # Column mapping
    "time_column": "date",
    "target_column": "sales",
    "exog_columns": ["temperature"],
    
    # Optional
    "parse_dates": ["date"],
    "frequency": "D"
}
```

### File Adapter

```python
{
    "type": "file",
    "path": "/path/to/data.csv",
    "format": "csv",  # csv, excel, parquet (auto-detected if not specified)
    
    # Column mapping
    "time_column": "date",
    "target_column": "sales",
    "exog_columns": ["temperature"],
    
    # CSV-specific options
    "csv_options": {
        "sep": ",",
        "header": 0,
        "encoding": "utf-8"
    },
    
    # Excel-specific options
    "excel_options": {
        "sheet_name": 0,
        "header": 0
    },
    
    # Common options
    "parse_dates": True,
    "frequency": "D"
}
```

## Data Validation

All data sources are automatically validated for:

- ✅ DatetimeIndex presence
- ✅ No duplicate time indices
- ✅ Sufficient data points
- ✅ Missing value detection
- ✅ Frequency inference

Validation results are included in the response:

```python
{
    "valid": True,
    "errors": [],  # Critical issues that prevent usage
    "warnings": ["Missing values detected: {'sales': 5.0}"]  # Non-critical issues
}
```

## Installation

### Core (Pandas support included)
```bash
pip install -e .
```

### With SQL support
```bash
pip install -e ".[sql]"
```

### With file format support
```bash
pip install -e ".[files]"
```

### With all optional dependencies
```bash
pip install -e ".[all]"
```

## Examples

See the `examples/` directory for complete working examples:

- `pandas_example.py` - Loading from pandas DataFrames
- `csv_example.py` - Loading from CSV/TSV files
- `sql_example.py` - Loading from SQL databases (SQLite, PostgreSQL, MySQL)

## Architecture

```
Data Source Layer
├── base.py                 # Abstract adapter interface
├── registry.py             # Adapter registry
└── adapters/
    ├── pandas_adapter.py   # In-memory DataFrames
    ├── sql_adapter.py      # SQL databases
    └── file_adapter.py     # CSV, Excel, Parquet
```

Each adapter implements:
- `load()` - Load data from source
- `validate()` - Check data quality
- `to_sktime_format()` - Convert to (y, X) format

## Metadata

Each loaded data source provides rich metadata:

```python
{
    "source": "sql",
    "rows": 100,
    "columns": ["sales", "temperature"],
    "frequency": "D",
    "start_date": "2020-01-01",
    "end_date": "2020-04-09",
    "missing_values": {"sales": 0, "temperature": 2}
}
```

## Error Handling

All operations return structured error information:

```python
{
    "success": False,
    "error": "Could not convert index to datetime",
    "error_type": "ValueError",
    "validation": {
        "valid": False,
        "errors": ["Index must be DatetimeIndex"]
    }
}
```

## Best Practices

1. **Always check validation results** before fitting models
2. **Release data handles** when done to free memory
3. **Use exogenous variables** when available for better forecasts
4. **Specify frequency** explicitly if auto-detection fails
5. **Handle missing values** before loading or use sktime's imputation

## Troubleshooting

### "No module named 'sqlalchemy'"
```bash
pip install sqlalchemy
```

### "No module named 'openpyxl'" (for Excel files)
```bash
pip install openpyxl
```

### "No module named 'pyarrow'" (for Parquet files)
```bash
pip install pyarrow
```

### "Could not infer frequency"
Specify frequency explicitly in config:
```python
config = {
    ...
    "frequency": "D"  # Daily, "W" for weekly, "M" for monthly, etc.
}
```

### "Index must be DatetimeIndex"
Ensure your time column is specified correctly:
```python
config = {
    ...
    "time_column": "date",  # Name of your date/time column
    "parse_dates": True
}
```
