Apache Beam pipeline that bulk-decompresses files in Google Cloud Storage. matches files by glob pattern, detects compression from the extension, decompresses to a target bucket, and logs failures to a CSV. skips files that already exist at the destination. ships in both Java and Python.

```bash
python cli_dataflow_decompress.py \
  --input_file_pattern=gs://my-bucket/*.gz \
  --output_bucket=gs://my-output-bucket \
  --output_failure_file=gs://my-bucket/failed.csv
```

[![python](https://img.shields.io/badge/python-3.8+-93450a.svg?style=flat-square)](https://www.python.org/)
[![java](https://img.shields.io/badge/java-beam_sdk-93450a.svg?style=flat-square)](https://beam.apache.org/)
[![license](https://img.shields.io/badge/license-MIT-grey.svg?style=flat-square)](https://opensource.org/licenses/MIT)

---

## what it does

- **glob matching** — point it at `gs://bucket/path/*.gz` and it finds everything
- **auto-detection** — figures out compression type from the file extension (GZIP, BZIP2, DEFLATE, ZIP)
- **idempotent** — checks if the output file already exists before decompressing, safe to re-run
- **dead-letter output** — uncompressed files, malformed archives, and I/O errors go to a CSV error log instead of crashing the pipeline
- **parallel decompression** (Python) — uses a thread pool sized to CPU count, processes files in configurable batches
- **strips extension** — `data.json.gz` becomes `data.json` at the destination

## supported compressions

| type | Java | Python |
|:---|:---|:---|
| GZIP | yes | yes |
| BZIP2 | yes | yes |
| DEFLATE | yes | yes |
| ZIP | yes | — |

detection is automatic via file extension. unrecognized extensions get routed to the error output.

## two implementations

### Java

designed as a Google Cloud Dataflow template (`CLI_Dataflow_Decompress`). uses `ValueProvider` for runtime parameter binding, Guava for byte copying, Apache Commons CSV for error output formatting.

```bash
java -cp <classpath> com.google.cloud.teleport.templates.CliDataflowDecompress \
  --inputFilePattern=gs://bucket/*.gz \
  --outputBucket=gs://output-bucket \
  --outputFailureFile=gs://bucket/failed.csv
```

### Python

standalone script. runs on Dataflow by default. uses `GcsIO` for streaming reads/writes and a `ThreadPoolExecutor` for parallel decompression within each worker.

```bash
python cli_dataflow_decompress.py \
  --input_file_pattern=gs://bucket/*.gz \
  --output_bucket=gs://output-bucket \
  --output_failure_file=gs://bucket/failed.csv \
  --batch_size=100
```

## configuration

### Java

| flag | required | description |
|:---|:---|:---|
| `--inputFilePattern` | yes | GCS glob pattern for input files |
| `--outputBucket` | yes | GCS bucket for decompressed output |
| `--outputFailureFile` | yes | GCS path for CSV error log |

plus standard Beam/Dataflow flags (`--runner`, `--project`, `--region`, etc.).

### Python

| flag | required | default | description |
|:---|:---|:---|:---|
| `--input_file_pattern` | yes | — | GCS glob pattern for input files |
| `--output_bucket` | yes | — | GCS bucket for decompressed output |
| `--output_failure_file` | yes | — | GCS path for CSV error log |
| `--batch_size` | no | `100` | files per batch |

plus standard Beam/Dataflow flags.

## output

decompressed files land in the output bucket with the same path structure, minus the compression extension.

failures go to a CSV:

```csv
Filename,Error
gs://bucket/bad-file.gz,"BZip2 format error: ..."
gs://bucket/not-compressed.txt,"file is not compressed"
```

## error handling

three cases are caught and routed to the dead-letter CSV instead of failing the pipeline:

- **uncompressed file** — no recognized compression extension
- **malformed archive** — BZip2 format errors, incorrect zlib headers
- **I/O errors** — general read/write failures

no explicit retry logic beyond Beam's built-in bundle retry on worker failure.

## dependencies

### Java

inferred from imports (no `pom.xml` in this repo — designed to live inside the [DataflowTemplates](https://github.com/GoogleCloudPlatform/DataflowTemplates) project):

- `org.apache.beam:beam-sdks-java-core`
- `com.google.guava:guava`
- `org.apache.commons:commons-csv`
- `org.slf4j:slf4j-api`

### Python

```
apache-beam[gcp]
google-cloud-storage
```

## project structure

```
CliDataflowDecompress.java    — Java implementation (Dataflow template)
cli_dataflow_decompress.py    — Python implementation (standalone)
```

Java key classes:
- `CliDataflowDecompress` — pipeline entry point and orchestration
- `CliDataflowDecompress.Decompress` — `DoFn` that handles per-file decompression logic
- `CliDataflowDecompress.Options` — pipeline options interface

Python key classes:
- `CliDataflowDecompressOptions` — CLI args and validation
- `Decompress` — `DoFn` with thread pool, handles decompression and GCS I/O
- `BatchElements` — `DoFn` that groups file metadata into batches

## license

MIT
