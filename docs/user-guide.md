# User Guide

This guide walks through prerequisites, installation, first-time use, and continuous usage patterns for sktime-mcp.

## Prerequisites

- Python 3.9+
- pip
- Optional extras based on your needs:
  - SQL databases: `.[sql]`
  - Excel/Parquet files: `.[files]`
  - Forecasting extras: `.[forecasting]`
  - Deep learning models: `.[dl]`

## Installation

```bash
pip install -e .
```

Optional extras:

```bash
pip install -e ".[sql]"
pip install -e ".[files]"
pip install -e ".[forecasting]"
pip install -e ".[dl]"
```

## Run The Server

```bash
sktime-mcp
```

or

```bash
python -m sktime_mcp.server
```

## Core Tooling (What The LLM Uses)

Discovery and inspection:

- `list_estimators` (filter by task and tags)
- `search_estimators` (search by name or docstring)
- `describe_estimator` (capabilities, hyperparameters)
- `get_available_tags` (list tag keys)

Instantiation and execution:

- `instantiate_estimator`
- `instantiate_pipeline`
- `validate_pipeline`
- `fit_predict`
- `fit` and `predict` (explicit lifecycle)
- `export_code` (reproducible Python)

Data loading and cleanup:

- `load_data_source`
- `list_data_sources`
- `fit_predict_with_data`
- `list_data_handles`
- `release_data_handle`

Data formatting:

- `format_time_series`
- `auto_format_on_load`

## First-Time Workflow (End-to-End)

1. Discover datasets

```json
{"tool": "list_datasets", "arguments": {}}
```

2. Find an estimator (example: forecasting)

```json
{"tool": "list_estimators", "arguments": {"task": "forecasting", "limit": 10}}
```

3. Inspect a candidate

```json
{"tool": "describe_estimator", "arguments": {"estimator": "NaiveForecaster"}}
```

4. Instantiate the estimator

```json
{
  "tool": "instantiate_estimator",
  "arguments": {"estimator": "NaiveForecaster", "params": {"strategy": "last", "sp": 12}}
}
```

5. Fit and predict using demo data

```json
{
  "tool": "fit_predict",
  "arguments": {"estimator_handle": "est_abc123", "dataset": "airline", "horizon": 12}
}
```

6. Export a reproducible code snippet

```json
{"tool": "export_code", "arguments": {"handle": "est_abc123", "var_name": "model"}}
```

## Working With Your Own Data

### Load From Pandas

```json
{
  "tool": "load_data_source",
  "arguments": {
    "config": {
      "type": "pandas",
      "data": {"date": ["2020-01-01"], "sales": [100]},
      "time_column": "date",
      "target_column": "sales"
    }
  }
}
```

### Load From CSV

```json
{
  "tool": "load_data_source",
  "arguments": {
    "config": {
      "type": "file",
      "path": "/path/to/data.csv",
      "time_column": "date",
      "target_column": "sales"
    }
  }
}
```

### Load From SQL

```json
{
  "tool": "load_data_source",
  "arguments": {
    "config": {
      "type": "sql",
      "connection_string": "postgresql://user:pass@host:5432/db",
      "query": "SELECT date, sales FROM sales",
      "time_column": "date",
      "target_column": "sales"
    }
  }
}
```

### Fit And Predict With Custom Data

```json
{
  "tool": "fit_predict_with_data",
  "arguments": {"estimator_handle": "est_abc123", "data_handle": "data_xyz789", "horizon": 7}
}
```

### Auto-Format and Data Cleaning

If you see validation warnings (missing values, irregular frequency), use:

```json
{
  "tool": "format_time_series",
  "arguments": {"data_handle": "data_xyz789", "auto_infer_freq": true, "fill_missing": true}
}
```

You can also enable auto-formatting on every load:

```json
{"tool": "auto_format_on_load", "arguments": {"enabled": true}}
```

## Continuous Usage Patterns

When running as a long-lived MCP service:

- Keep one or a few estimator handles for repeated runs
- Release unused handles to avoid memory growth
- Keep data handles for active datasets only
- Use `export_code` to persist a successful configuration outside of the LLM session

Handle lifecycle:

- `list_handles` and `release_handle` for estimators
- `list_data_handles` and `release_data_handle` for data

## Common Errors And Fixes

- `Unknown estimator`: use `search_estimators` or `list_estimators`
- `Unknown dataset`: use `list_datasets`
- Missing SQL or file modules: install `.[sql]` or `.[files]`
- Validation warnings: use `format_time_series` or `auto_format_on_load`
