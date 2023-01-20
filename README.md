<h1 align="center">Target-BigQuery</h1>

<p align="center">
<a href="https://github.com/z3z1ma/target-bigquery/actions/"><img alt="Actions Status" src="https://github.com/z3z1ma/target-bigquery/actions/workflows/ci.yml/badge.svg"></a>
<a href="https://github.com/z3z1ma/target-bigquery/blob/main/LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-yellow.svg"></a>
<a href="https://github.com/psf/black"><img alt="Code style: black" src="https://img.shields.io/badge/code%20style-black-000000.svg"></a>
</p>

**A rare 💎 you have stumbled upon**

`target-bigquery` is a Singer target for BigQuery.

It is the most versatile target for BigQuery. Extremely performant, resource efficient, and fast in all configurations enabling over 8 different ingestion patterns. Denormalized variants indicate data is unpacked during load with a resultant schema in BigQuery based on the tap schema. Non-denormalized means we have a fixed schema which loads all data into an unstructured `JSON` column. They are both useful patterns. The latter allowing BigQuery to work with schemaless or rapidly changing sources such as MongoDB instantly, while the former is more performant and convenient to start modeling quickly.

## Installation 📈

The package on pypi is named `z3-target-bigquery` but the executable it ships with is simply `target-bigquery`. This allows me to release work without concerns of naming conflicts on the package index.

```bash
# Use pipx or pip
pipx install z3-target-bigquery
# Verify it is installed
target-bigquery --version
```

## Features ✨

- Autoscaling self-healing worker pool using either threads (default) or multiprocessing, configurable by the user for the _fastest_ possible data ingestion. Particularly when leveraging colocated compute in GCP.
- Denormalized load pattern where data is unpacked in flight into a statically typed BigQuery schema derived from the input stream json schemas.
- Fix schema load pattern where all data is loaded into a `JSON` column which has been GA in BigQuery since mid 2022.
- Autogenerated `VIEW` support for fixed schema load patterns which essentially overlays a statically typed schema allowing you to get the best of both worlds when using fixed schema ingestion.
- JIT compilation of protobuf schemas allowing the Storage Write API to use a denormalized load pattern. 
- BATCH message support 😎

## Load Patterns 🏎

- `Batch Load Job` ingestion pattern using in memory compression (fixed schema + denormalized)
- `Storage Write API` ingestion pattern using gRPC and protocol buffers supporting both streaming and batch patterns. Capable of JIT compilation of BQ schemas to protobuf to enable denormalized loads of input structures only known at runtime. (fixed schema + denormalized 🎉)
- `GCS Staging` ingestion pattern using in memory compression and a GCS staging layer which generates a well organized data lake which backs the data warehouse providing additional failsafes and data sources (fixed schema + denormalized)
- `Legacy Streaming` ingestion pattern which emphasizes simplicity and fast start up / tear down. I would highly recommend the storage write API instead unless data volume is small (fixed schema + denormalized)


**Choosing between denormalized and fixed schema (JSON support)?** 🙇🏾‍♂️

The gap between the methods is closed due in part to the target's ability to  automatically generating a `VIEW` which will unpack (or rather provide typing as a more accurate take) a JSON based ingestion source for you. Unless operating at tens of millions of rows with JSON objects containing multiple hundreds of keys, its quite performant. Particularly if accessing a small subset of keys. It does however fall off (quite hard) given enough scale as I mentioned (I've pushed it to the limits). Denormalized is recommended for high volume where the schema is fairly consistent. Fixed is recommended for lower volume, inconsistent schemas or for taps which are inherently schemaless in which case its the ideal (only...) logical pattern. Fixed schema can also be used for taps which routinely break down on BQ due to json schemas being inexpressible in a static way (ie patternProperties, additionalProperties...)


**Old Header (still true, here for posterity)** 🧪

This is the first truly unstructured sink for BigQuery leveraging the recent GA feature in BigQuery for JSON support. This allows this target to load from essentially any tap regardless of the quality or explicitness of its jsonschema. Observations in existing taps note things such as `patternProperties` used in jsonschema objects which break down on all existing BigQuery taps due to the previous need for strong typing. Also taps such as MongoDB which inherently deal with unstructured data are seamlessly enabled by this target without klutzy collection scraping of a sample of records which we _hope_ are repesentative of all documents.


Built with the [Meltano Target SDK](https://sdk.meltano.com).


## Configuration 🔨

### Settings

First a valid example to give context to the below including a nested key example (denoted via a `.` in the setting path) as seen with `column_name_transforms.snake_case`

```json
{
    "project": "my-bq-project",
    "method": "storage_write_api",
    "denormalized": true,
    "credentials_path": "...",
    "dataset": "my_dataset",
    "batch_size": 500,
    "column_name_transforms": {
      "snake_case": true
    }
}
```


| Setting             | Required | Default | Description |
|:--------------------|:--------:|:-------:|:------------|
| credentials_path    | False    | None    | The path to a gcp credentials json file. |
| credentials_json    | False    | None    | A JSON string of your service account JSON file. |
| project             | True     | None    | The target GCP project to materialize data into. |
| dataset             | True     | None    | The target dataset to materialize data into. |
| batch_size          | False    |  500    | The maximum number of rows to send in a single batch to the worker. This should be configured based on load method. For `storage_write_api` and `streaming_insert` it should be `<=500`, for the LoadJob sinks, it can be much higher, ie `>100,000` |
| timeout             | False    |     600 | Default timeout for batch_job and gcs_stage derived LoadJobs. |
| denormalized        | False    |       0 | Determines whether to denormalize the data before writing to BigQuery. A false value will write data using a fixed JSON column based schema, while a true value will write data using a dynamic schema derived from the tap. |
| method              | True     | storage_write_api | The method to use for writing to BigQuery. Must be one of `batch_job`, `storage_write_api`, `gcs_stage`, `streaming_insert` |
| generate_view       | False    |       0 | Determines whether to generate a view based on the SCHEMA message parsed from the tap. Only valid if denormalized=false meaning you are using the fixed JSON column based schema. |
| bucket              | False    | None    | The GCS bucket to use for staging data. Only used if method is gcs_stage. |
| cluster_on_key_properties| False    | False    | Indicates if we should use the key_properties from the tap to cluster our table. By default, tables created by this target cluster on `_sdc_batched_at`. |
| partition_granularity| False    | "month"    | Indicates the granularity of the created table partitioning scheme which is based on `_sdc_batched_at`. By default the granularity is monthly. Must be one of: "hour", "day", "month", "year". |
| column_name_transforms.lower| False    | None    | Lowercase column names. |
| column_name_transforms.quote| False    | None    | Quote column names in any generated DDL. |
| column_name_transforms.add_underscore_when_invalid| False    | None    | Add an underscore to the column name if it starts with a digit to make it valid. |
| column_name_transforms.snake_case| False    | None    | Snake case all incoming column names. Does not apply to fixed schema loads but _does_ apply to the view auto-generated over them. |
| options.storage_write_batch_mode| False    | None    | By default, we use the default stream (Committed mode) in the storage_write_api load method which results in streaming records which are immediately available and is generally fastest. If this is set to true, we will use the application created streams (Committed mode) to transactionally batch data on STATE messages and at end of pipe. |
| options.process_pool | False    | None    | By default we use an autoscaling threadpool to write to BigQuery. If set to true, we will use a process pool. |
| options.max_workers_per_stream| False    | None    | By default, each sink type has a preconfigured max worker limit. This sets an override for maximum number of workers per stream. |
| stream_maps         | False    | None    | Config object for stream maps capability. For more information check out [Stream Maps](https://sdk.meltano.com/en/latest/stream_maps.html). |
| stream_map_config   | False    | None    | User-defined config values to be used within map expressions. |
| flattening_enabled  | False    | None    | 'True' to enable schema flattening and automatically expand nested properties. |
| flattening_max_depth| False    | None    | The max depth to flatten schemas. |

A full list of supported settings and capabilities is available by running: `target-bigquery --about`

### Configure using environment variables ✏️

This Singer target will automatically import any environment variables within the working directory's
`.env` if the `--config=ENV` is provided, such that config values will be considered if a matching
environment variable is set either in the terminal context or in the `.env` file.

### Source Authentication and Authorization 👮🏽‍♂️

Authenticate via service account key file or Application Default Credentials (ADC)
https://cloud.google.com/bigquery/docs/authentication

## Capabilities ✨

* `about`
* `stream-maps`
* `schema-flattening`
* `batch`

## Usage 👷‍♀️

You can easily run `target-bigquery` by itself or in a pipeline using [Meltano](https://meltano.com/).


### Executing the Target Directly 🚧

```bash
target-bigquery --version
target-bigquery --help
# Test using the "Carbon Intensity" sample:
tap-carbon-intensity | target-bigquery --config /path/to/target-bigquery-config.json
```

## Developer Resources 👩🏼‍💻


### Initialize your Development Environment

```bash
pipx install poetry
poetry install
```

### Create and Run Tests

Create tests within the `target_bigquery/tests` subfolder and
  then run:

```bash
poetry run pytest
```

You can also test the `target-bigquery` CLI interface directly using `poetry run`:

```bash
poetry run target-bigquery --help
```

### Testing with [Meltano](https://meltano.com/)

_**Note:** This target will work in any Singer environment and does not require Meltano.
Examples here are for convenience and to streamline end-to-end orchestration scenarios._

Next, install Meltano (if you haven't already) and any needed plugins:

```bash
# Install meltano
pipx install meltano
# Initialize meltano within this directory
cd target-bigquery
meltano install
```

Now you can test and orchestrate using Meltano:

```bash
# Test invocation:
meltano invoke target-bigquery --version
# OR run a test `elt` pipeline with the Carbon Intensity sample tap:
meltano elt tap-carbon-intensity target-bigquery
```

### SDK Dev Guide

See the [dev guide](https://sdk.meltano.com/en/latest/dev_guide.html) for more instructions on how to use the Meltano SDK to
develop your own Singer taps and targets.
