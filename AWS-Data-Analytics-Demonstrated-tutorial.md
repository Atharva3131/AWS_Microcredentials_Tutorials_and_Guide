# AWS IoT Fleet Intelligence: Serverless Telemetry Analytics

A reference implementation for collecting vehicle and equipment
telemetry, validating events, storing them in an Amazon S3 data lake,
querying fleet health with Amazon Athena, and publishing operational
dashboards with Amazon QuickSight.

> **Scope:** This is an independently designed educational project. It
> uses a fictional fleet operator and synthetic telemetry. It is not a
> production-ready fleet-control system and does not issue commands to
> vehicles or equipment.

## 1. Introduction

Fleet operators need to understand equipment utilization, location
freshness, temperature excursions, battery condition, and fault
frequency across a distributed fleet. Device data arrives continuously,
varies in quality, and can become expensive to retain and query without
a deliberate data strategy.

This tutorial demonstrates a serverless analytics pattern built around
managed AWS services:

1.  Devices publish telemetry through MQTT.
2.  AWS IoT Core evaluates messages against a topic rule.
3.  A lightweight AWS Lambda function validates and enriches events.
4.  Amazon Data Firehose buffers records and delivers them to Amazon S3.
5.  AWS Glue Data Catalog describes the stored data.
6.  Amazon Athena supports ad-hoc SQL analysis.
7.  Amazon QuickSight presents fleet-level operational metrics.
8.  Amazon CloudWatch provides logs, metrics, and alarms.

The design separates ingestion, storage, cataloging, analysis, and
visualization so each layer can evolve independently.

### Learning objectives

After completing the tutorial, you should be able to:

-   Explain an event-driven IoT ingestion pipeline.
-   Design a versioned telemetry event contract.
-   Validate and enrich incoming JSON events.
-   Build an S3 data lake with time-based partitions.
-   Catalog data and query it with Athena.
-   Create operational metrics and dashboards.
-   Apply least-privilege access, encryption, retention, and monitoring.
-   Estimate and optimize the cost of a serverless analytics workload.

## 2. Prerequisites

-   An AWS account in a region that supports the services used.
-   Familiarity with JSON, Python 3, SQL, and basic IAM concepts.
-   AWS CLI v2 configured with a named profile.
-   An MQTT client or a small device simulator.
-   An S3 bucket dedicated to this project.
-   Permission to create IoT Core rules, Lambda functions, Firehose
    delivery streams, Glue catalog resources, Athena workgroups, and
    QuickSight datasets.

Use placeholders for account-specific values. Never commit access keys,
private certificates, passwords, or account identifiers to source
control.

Example environment variables:

``` bash
export AWS_REGION="us-east-1"
export PROJECT_PREFIX="fleet-intelligence-demo"
export DATA_BUCKET="replace-with-a-globally-unique-bucket-name"
export FIREHOSE_STREAM="fleet-telemetry-delivery"
```

## 3. Reference architecture

![AWS IoT Fleet Intelligence reference architecture](docs/AWS-Data-Analytics%20%20Architecture.png)



### Component responsibilities

  -----------------------------------------------------------------------
  Layer                   AWS service             Responsibility
  ----------------------- ----------------------- -----------------------
  Device connectivity     AWS IoT Core            Authenticated MQTT
                                                  ingestion and
                                                  topic-based routing

  Event processing        AWS Lambda              Schema checks,
                                                  normalization, and
                                                  rejection of malformed
                                                  events

  Delivery                Amazon Data Firehose    Buffering and managed
                                                  delivery to object
                                                  storage

  Data lake               Amazon S3               Durable storage of raw
                                                  and normalized
                                                  telemetry

  Metadata                AWS Glue Data Catalog   Table and partition
                                                  metadata for analytics

  Query                   Amazon Athena           SQL analysis over S3
                                                  data

  Visualization           Amazon QuickSight       Operational dashboards
                                                  and scheduled reporting

  Observability           Amazon CloudWatch       Logs, service metrics,
                                                  alarms, and operational
                                                  visibility

  Encryption              AWS KMS                 Customer-managed
                                                  encryption keys where
                                                  required
  -----------------------------------------------------------------------

### Data flow

1.  A device publishes a JSON event to a topic such as
    `demo/fleet/vehicle-042/telemetry`.
2.  An IoT rule forwards eligible messages to a Lambda function.
3.  Lambda validates required fields, normalizes units, and sends
    accepted records to Firehose.
4.  Firehose buffers records and writes objects to S3 using a
    date-partitioned prefix.
5.  A Glue crawler or managed table definition makes the data
    discoverable.
6.  Athena queries the data; QuickSight consumes curated query results.

For high-volume deployments, consider a direct IoT Core-to-Firehose rule
action or a stream-based fan-out design. Choose the processing hop based
on validation complexity, throughput, latency, and cost requirements.

## 4. Core technical concepts

### 4.1 Telemetry event contract

A stable event contract is more important than any individual dashboard.
Include an explicit schema version, stable device identifier, event
timestamp, units, and a unique event identifier.

Example event:

``` json
{
  "schema_version": 1,
  "event_id": "evt-7f31a9",
  "device_id": "vehicle-042",
  "event_time": "2026-10-18T09:42:15Z",
  "latitude": 12.9716,
  "longitude": 77.5946,
  "speed_kph": 48.2,
  "engine_temp_c": 91.4,
  "battery_voltage_v": 12.6,
  "fuel_level_pct": 67.0,
  "odometer_km": 18452.7,
  "fault_code": null
}
```

Use UTC timestamps and document whether each measurement is
instantaneous, cumulative, or derived. Do not silently change units or
field meanings between schema versions.

### 4.2 Data lake zones

A simple layout separates original input from normalized analytical
data:

``` text
s3://<data-bucket>/
  raw/
    telemetry/
      ingest_year=2026/ingest_month=10/ingest_day=18/
  curated/
    telemetry/
      event_year=2026/event_month=10/event_day=18/
  rejected/
    telemetry/
```

-   **Raw:** immutable or append-only source records for replay and
    audit.
-   **Curated:** validated, normalized records optimized for analysis.
-   **Rejected:** malformed or unsupported events with restricted access
    and a defined retention period.

Partition by frequently filtered dimensions such as event date. Avoid
excessive partition cardinality, such as partitioning by individual
device ID when the fleet is large.

### 4.3 Operational metrics

Useful fleet metrics include:

-   **Reporting freshness:** current time minus the latest event time
    per device.
-   **Fault-event rate:** fault events divided by total events over a
    defined window.
-   **Temperature excursion count:** events above an engineering-defined
    threshold.
-   **Utilization:** active operating time divided by available time.
-   **Data completeness:** received expected events divided by expected
    events.

Thresholds must be defined by the equipment manufacturer or operational
policy. Example values in a demo should not be treated as safety limits.

## 5. Implementation

This section provides a compact implementation path. Adapt names, IAM
policies, retention, and service quotas to your account and workload.

### 5.1 Create storage and delivery resources

1.  Create a private S3 bucket for the data lake.
2.  Enable S3 Block Public Access.
3.  Enable bucket versioning if recovery from accidental changes is
    required.
4.  Configure default encryption with SSE-S3 or SSE-KMS according to
    your security requirements.
5.  Create a Firehose delivery stream with S3 as its destination.
6.  Configure buffering and an S3 prefix such as `raw/telemetry/`.
7.  Enable failed-delivery logging and configure an error-output prefix.

A production delivery stream should have a dedicated IAM role that can
write only to the intended bucket and prefixes. Avoid wildcard access to
all S3 buckets.

### 5.2 Validate and forward events with Lambda

The following example is intended for an IoT rule invoking Lambda with a
JSON event. It validates required fields and forwards accepted events to
Firehose. Configure the `DELIVERY_STREAM` environment variable and grant
the function only `firehose:PutRecord` on the selected delivery stream.

``` python
import json
import os
from datetime import datetime, timezone

import boto3

firehose = boto3.client("firehose")
DELIVERY_STREAM = os.environ["DELIVERY_STREAM"]

REQUIRED_FIELDS = {
    "schema_version",
    "event_id",
    "device_id",
    "event_time",
    "speed_kph",
    "engine_temp_c",
}

def parse_utc(value: str) -> datetime:
    parsed = datetime.fromisoformat(value.replace("Z", "+00:00"))
    if parsed.tzinfo is None:
        raise ValueError("event_time must include a timezone")
    return parsed.astimezone(timezone.utc)

def lambda_handler(event, context):
    missing = REQUIRED_FIELDS.difference(event)
    if missing:
        raise ValueError(f"Missing required fields: {sorted(missing)}")

    if event["schema_version"] != 1:
        raise ValueError("Unsupported schema_version")

    if not isinstance(event["device_id"], str) or not event["device_id"]:
        raise ValueError("device_id must be a non-empty string")

    event_time = parse_utc(event["event_time"])

    normalized = {
        **event,
        "event_time": event_time.isoformat().replace("+00:00", "Z"),
        "ingested_at": datetime.now(timezone.utc).isoformat().replace("+00:00", "Z"),
    }

    response = firehose.put_record(
        DeliveryStreamName=DELIVERY_STREAM,
        Record={"Data": (json.dumps(normalized) + "\n").encode("utf-8")},
    )

    return {
        "accepted": True,
        "record_id": response["RecordId"],
    }
```

**Operational notes**

-   Lambda retries and IoT rule delivery behavior should be understood
    before production use.
-   Design for duplicate delivery. Use `event_id` as an idempotency key
    in downstream processing.
-   Add numeric range checks, payload-size limits, schema compatibility
    checks, and structured error handling.
-   Do not log full telemetry payloads if they contain sensitive
    location or customer information.
-   For sustained high throughput, batch records and evaluate a
    stream-based processing design instead of invoking Lambda once per
    event.

### 5.3 Configure the IoT rule

Create an IoT Core rule that matches the project's device topic
namespace. The rule should:

-   Select only the expected telemetry topic pattern.
-   Validate or filter the schema version where practical.
-   Invoke the Lambda function.
-   Use a narrowly scoped role or Lambda resource permission.
-   Route failures to a monitored recovery path.

Keep device certificates unique. Do not share a single device identity
across an entire fleet.

### 5.4 Catalog the S3 data

Create a Glue database for the project and use a crawler or explicit
table definition to catalog the curated telemetry prefix. Confirm that
timestamp and numeric fields have the intended types.

For repeatable production pipelines, prefer schema-controlled table
definitions and automated schema evolution checks over relying
exclusively on crawler inference.

### 5.5 Query telemetry with Athena

Example query for daily fleet temperature summaries. Adjust the table
and column names to match the Glue catalog definition.

``` sql
SELECT
    date_trunc('day', event_timestamp) AS event_day,
    COUNT(*) AS event_count,
    COUNT(DISTINCT device_id) AS reporting_devices,
    AVG(engine_temp_c) AS avg_engine_temp_c,
    MAX(engine_temp_c) AS max_engine_temp_c
FROM fleet_analytics.telemetry_curated
WHERE event_date >= DATE '2026-10-01'
  AND event_date < DATE '2026-11-01'
GROUP BY 1
ORDER BY 1;
```

Example query to identify devices that have not reported recently:

``` sql
SELECT
    device_id,
    MAX(event_timestamp) AS last_seen
FROM fleet_analytics.telemetry_curated
WHERE event_date >= current_date - INTERVAL '2' DAY
GROUP BY device_id
ORDER BY last_seen ASC;
```

Use a bounded date predicate so Athena can skip irrelevant partitions.
Configure an Athena workgroup with an output location, encryption, and
per-query data-scan controls.

### 5.6 Build the dashboard

Create a QuickSight dataset from the curated Athena table or a saved
Athena query. Useful visuals include:

-   Active reporting devices by day.
-   Average and maximum engine temperature over time.
-   Fault-event count by device or region.
-   Device reporting freshness.
-   Fleet utilization by vehicle class.

Use row-level security when users should see only assigned fleets or
regions. Validate access with test identities before sharing dashboards.

## 6. Security and reliability

### Identity and access

-   Use separate IAM roles for IoT rules, Lambda, Firehose, Glue,
    Athena, and QuickSight access.
-   Apply least privilege to resource ARNs and S3 prefixes.
-   Use AWS IAM Identity Center or an approved federated identity
    provider for human access.
-   Rotate device certificates and revoke compromised identities
    promptly.

### Data protection

-   Keep S3 buckets private and enable Block Public Access.
-   Encrypt data at rest and in transit.
-   Use KMS key policies that limit decrypt access to required roles.
-   Define retention for raw, curated, and rejected data.
-   Restrict access to precise location data and apply retention
    minimization.

### Reliability

-   Monitor Lambda errors, throttles, duration, and concurrency.
-   Monitor Firehose delivery failures and data freshness.
-   Configure alarms for ingestion gaps and sustained processing errors.
-   Use dead-letter or quarantine paths where supported by the selected
    integration.
-   Test replay procedures and duplicate-event handling.
-   Document recovery objectives and verify backups or versioning
    against them.

## 7. Testing and troubleshooting

### Test strategy

  -----------------------------------------------------------------------
  Test                                Expected result
  ----------------------------------- -----------------------------------
  Valid telemetry event               Accepted and delivered to the data
                                      lake

  Missing required field              Rejected or quarantined with a
                                      traceable error

  Unsupported schema version          Rejected without corrupting curated
                                      data

  Duplicate event ID                  Duplicate detected or safely
                                      tolerated downstream

  Delayed event timestamp             Stored with event time and
                                      ingestion time preserved

  Firehose delivery interruption      Failure visible through service
                                      metrics and logs

  Unauthorized device identity        MQTT publish denied

  Unauthorized dashboard user         Restricted rows and datasets remain
                                      inaccessible
  -----------------------------------------------------------------------

### Common failure modes

  -----------------------------------------------------------------------
  Symptom                             Investigation
  ----------------------------------- -----------------------------------
  No messages reach processing        Check MQTT topic, device
                                      certificate policy, and IoT rule
                                      metrics

  Lambda invocation fails             Inspect CloudWatch logs, execution
                                      role, timeout, and input shape

  Firehose cannot write to S3         Verify delivery role, bucket
                                      policy, KMS permissions, and
                                      destination prefix

  Glue table has incorrect types      Review source JSON, crawler
                                      configuration, and schema
                                      definition

  Athena query scans too much data    Add partition predicates and use
                                      columnar curated formats

  QuickSight shows stale data         Check dataset refresh mode, Athena
                                      query output, and ingestion
                                      freshness

  Duplicate rows appear               Check delivery semantics and
                                      downstream idempotency using
                                      `event_id`
  -----------------------------------------------------------------------

## 8. Cost and performance considerations

-   **IoT Core:** control message frequency and payload size; avoid
    sending unchanged state repeatedly.
-   **Lambda:** batch work where possible and select memory based on
    measured duration and throughput.
-   **Firehose:** tune buffering for the latency-versus-object-size
    trade-off.
-   **S3:** use lifecycle policies to transition or expire data
    according to retention requirements.
-   **Athena:** partition by event date, convert curated data to
    Parquet, compress files, and avoid tiny objects.
-   **QuickSight:** select SPICE or direct query based on freshness,
    concurrency, and cost requirements.
-   **CloudWatch:** set log retention and avoid high-cardinality custom
    metrics unless they provide operational value.

Measure actual traffic, query scans, dashboard concurrency, and
retention before estimating monthly cost. Use the AWS Pricing Calculator
for region-specific estimates.

## 9. Further improvements

Possible extensions include:

-   Schema registry and automated contract testing.
-   Apache Parquet conversion using AWS Glue ETL.
-   Near-real-time anomaly detection with managed streaming analytics or
    a dedicated ML workflow.
-   Geospatial analysis of route and service-area patterns.
-   Data quality checks for missing, late, and out-of-range
    measurements.
-   Infrastructure as Code using AWS CDK, CloudFormation, or Terraform.
-   CI/CD pipelines with unit tests, integration tests, and security
    scanning.

## 10. Official documentation

-   [AWS IoT Core Developer
    Guide](https://docs.aws.amazon.com/iot/latest/developerguide/what-is-aws-iot.html)
-   [AWS IoT
    rules](https://docs.aws.amazon.com/iot/latest/developerguide/iot-rules.html)
-   [AWS Lambda Developer
    Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
-   [Amazon Data Firehose Developer
    Guide](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html)
-   [Amazon S3 User
    Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
-   [AWS Glue Data
    Catalog](https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html)
-   [Amazon Athena User
    Guide](https://docs.aws.amazon.com/athena/latest/ug/what-is.html)
-   [Amazon QuickSight User
    Guide](https://docs.aws.amazon.com/quicksight/latest/user/welcome.html)
-   [Amazon CloudWatch User
    Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
-   [AWS Well-Architected
    Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
-   [AWS Pricing Calculator](https://calculator.aws/)

## 11. Conclusion

This project illustrates how managed AWS services can form a modular
telemetry analytics platform. The key engineering decisions are the
event contract, validation boundary, partition strategy, least-privilege
permissions, failure handling, and query-cost controls. Start with a
small synthetic workload, measure each stage, and introduce additional
services only when requirements justify the operational complexity.

------------------------------------------------------------------------

**Educational use:** All fleet entities and sample records in this
tutorial are fictional. Validate service limits, regional availability,
security requirements, and costs against current AWS documentation
before deploying.
