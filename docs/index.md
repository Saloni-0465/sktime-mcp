# sktime-mcp Documentation

sktime-mcp is a Model Context Protocol (MCP) server that exposes sktime's time-series estimators to LLM clients. It provides discovery, composition, and execution tools so an LLM can find the right estimator, validate pipelines, and run real forecasts on real data.

## What You Can Do

- Discover estimators by task and tags
- Inspect estimator capabilities and hyperparameters
- Instantiate estimators and pipelines
- Fit and predict on demo datasets or your own data
- Export Python code for reproducibility
- Load data from pandas, files, or SQL databases

## Prerequisites

- Python 3.9 or newer
- pip
- Optional extras depending on your data sources:
  - SQL: `.[sql]`
  - Files (Excel/Parquet): `.[files]`
  - Forecasting extras: `.[forecasting]`
  - Deep learning: `.[dl]`

## Install

From the repo root:

```bash
pip install -e .
```

Common options:

```bash
# All optional dependencies
pip install -e ".[all]"

# Development tools
pip install -e ".[dev]"
```

## Start The MCP Server

```bash
# Entry point
sktime-mcp

# Or
python -m sktime_mcp.server
```

The server uses stdio transport, which is compatible with MCP clients like Claude Desktop.

## First-Time Use (Quick Walkthrough)

These are the typical first steps a client should take. The examples show tool arguments, which MCP clients send as JSON in tool calls.

1. List demo datasets

```json
{
  "tool": "list_datasets",
  "arguments": {}
}
```

2. Discover forecasting estimators

```json
{
  "tool": "list_estimators",
  "arguments": {"task": "forecasting", "limit": 10}
}
```

3. Inspect a specific estimator

```json
{
  "tool": "describe_estimator",
  "arguments": {"estimator": "NaiveForecaster"}
}
```

4. Instantiate an estimator

```json
{
  "tool": "instantiate_estimator",
  "arguments": {
    "estimator": "NaiveForecaster",
    "params": {"strategy": "last", "sp": 12}
  }
}
```

5. Fit and predict on a demo dataset

```json
{
  "tool": "fit_predict",
  "arguments": {
    "estimator_handle": "est_abc123",
    "dataset": "airline",
    "horizon": 12
  }
}
```

6. Export reproducible Python code

```json
{
  "tool": "export_code",
  "arguments": {"handle": "est_abc123", "var_name": "model"}
}
```

## Continuous Use Patterns

When you keep the server running as a long-lived MCP service, you will usually:

- Reuse estimator handles for repeated forecasting runs
- Release estimator and data handles to avoid memory growth
- Use `auto_format_on_load` for stable ingestion of messy data
- Export code after a successful workflow to make results reproducible

Handle management tools:

- `list_handles` and `release_handle` for estimator lifecycle
- `list_data_handles` and `release_data_handle` for data lifecycle

## Examples

The `examples/` directory provides runnable scripts that showcase different workflows:

- `examples/01_forecasting_workflow.py` (full discovery → fit → predict)
- `examples/03_pipeline_instantiation.py` (pipeline construction)
- `examples/04_mcp_pipeline_demo.py` (MCP-style pipeline flow)
- `examples/pandas_example.py`, `examples/csv_example.py`, `examples/sql_example.py`

Run any example from the repo root:

```bash
python examples/01_forecasting_workflow.py
```

## Troubleshooting

- Missing SQL or file dependencies:

```bash
pip install -e ".[sql]"
pip install -e ".[files]"
```

- Unknown estimator or dataset:
  - Use `list_estimators` or `list_datasets` to discover valid names

- Data validation warnings:
  - Use `format_time_series` or enable `auto_format_on_load`

For detailed usage, continue to the User Guide.
