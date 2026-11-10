# Building an End-to-End MLOps Workflow on AWS

**A practical guide to model development, training, registration, deployment, inference, and monitoring.**

> An independently designed MLOps reference project using a predictive maintenance use case. This tutorial explains general AWS engineering concepts and does not reproduce certification assessment tasks or architecture.

## 1. Introduction

Machine learning models require more than experimentation and training. A production ML system must manage datasets, model versions, deployment, inference, monitoring, and retraining.

This tutorial develops an MLOps workflow for a fictional industrial equipment company.

The company collects equipment telemetry and wants to predict the probability of equipment failure. The platform must support repeatable model training, controlled model releases, scalable inference, and operational monitoring.

### Learning objectives

- Understand the MLOps lifecycle.
- Use Amazon SageMaker for model development and training.
- Store datasets and model artifacts in Amazon S3.
- Manage model container images with Amazon ECR.
- Register and version models.
- Deploy a real-time SageMaker inference endpoint.
- Expose inference through API Gateway and Lambda.
- Monitor application and model performance.
- Apply security, reliability, and cost optimization practices.

## 2. Prerequisites

- AWS account with appropriate service permissions.
- Python 3.10+ and basic ML knowledge.
- AWS CLI v2.
- Familiarity with Docker and container images.
- Basic knowledge of IAM, S3, and REST APIs.
- Understanding of classification and model evaluation.

Use a dedicated development environment and establish an AWS budget before provisioning resources.


## 3. Independently Designed Reference Architecture

### Architecture diagram

![AWS MLOps architecture blueprint](docs/AWS%20MLOps%20Architecture%20Blueprint.png)

### Use case: Predictive maintenance

Industrial equipment produces telemetry such as:

- Operating temperature.
- Vibration level.
- Operating hours.
- Pressure.
- Maintenance history.

A classification model estimates the likelihood of equipment failure within a defined prediction window.

### Architecture components

| Component | Responsibility |
|---|---|
| Amazon S3 | Store training data, validation data, and model artifacts |
| SageMaker Studio | Model experimentation and development |
| SageMaker Processing | Data validation and preprocessing |
| SageMaker Training | Train and evaluate ML models |
| SageMaker Pipelines | Orchestrate repeatable ML workflows |
| SageMaker Model Registry | Version and approve model releases |
| Amazon ECR | Store container images used by ML workloads |
| SageMaker Endpoint | Host the deployed model |
| AWS Lambda | Invoke model inference |
| Amazon API Gateway | Expose a REST inference interface |
| Amazon CloudWatch | Monitor infrastructure and application health |
| SageMaker Model Monitor | Detect data-quality and model-quality issues |

### End-to-end lifecycle

1. Upload and validate datasets.
2. Prepare features and training labels.
3. Train a model.
4. Evaluate model quality against defined criteria.
5. Register the approved model version.
6. Deploy the model to a managed endpoint.
7. Expose inference through a REST API.
8. Monitor latency, errors, and data quality.
9. Retrain and promote improved model versions.

## 4. Core MLOps Concepts

### 4.1 Data versioning

Training datasets must be traceable to the model versions they produce.

Store datasets in structured S3 prefixes, for example:

```text
s3://ml-platform-data/
├── datasets/
│   ├── equipment-v1/
│   └── equipment-v2/
├── processed/
├── models/
└── evaluation/
```

Use S3 versioning and record dataset identifiers, preprocessing versions, and training parameters.

### 4.2 Feature engineering

Raw sensor measurements are often unsuitable for direct model training.

Example features:

- Average temperature over a defined time window.
- Maximum vibration level.
- Operating hours since maintenance.
- Temperature change rate.
- Vibration variability.

Feature engineering should be consistent between training and inference to prevent training-serving skew.

### 4.3 Model training

A supervised classification model learns from labeled historical equipment records.

A simple training example using XGBoost:

```python
import pandas as pd
from xgboost import XGBClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score

data = pd.read_csv("equipment_training.csv")

X = data.drop(columns=["failure_within_window"])
y = data["failure_within_window"]

X_train, X_test, y_train, y_test = train_test_split(
    X,
    y,
    test_size=0.2,
    random_state=42,
    stratify=y,
)

model = XGBClassifier(
    n_estimators=200,
    max_depth=5,
    learning_rate=0.05,
    eval_metric="logloss",
)

model.fit(X_train, y_train)

probabilities = model.predict_proba(X_test)[:, 1]

print("ROC-AUC:", roc_auc_score(y_test, probabilities))
```

The dataset and target label are illustrative. A real deployment requires careful handling of class imbalance, temporal leakage, and the business cost of false positives and false negatives.

### 4.4 Model registry

A model registry provides a controlled record of trained model versions.

A typical release process is:

```text
Training completed
       |
       v
Model evaluation
       |
       v
Quality threshold passed?
       |
       +---- No ----> Reject model
       |
      Yes
       |
       v
Register model version
       |
       v
Approval workflow
       |
       v
Deploy to production
```

Approval should be based on measurable acceptance criteria, not merely successful training.

### 4.5 Model deployment

A SageMaker real-time endpoint hosts a model for low-latency inference.

Deployment considerations include:

- Instance selection.
- Autoscaling.
- Health checks.
- Endpoint configuration versions.
- Rollback procedures.
- Request and response payload limits.

### 4.6 Inference API

A REST API provides a stable interface for applications that need predictions.

Example request:

```json
{
  "equipment_id": "machine-104",
  "temperature": 72.4,
  "vibration": 0.83,
  "operating_hours": 1820,
  "pressure": 4.7
}
```

Example response:

```json
{
  "equipment_id": "machine-104",
  "failure_probability": 0.78,
  "risk_category": "high",
  "model_version": "v3"
}
```

These values are illustrative and do not represent a trained production model.

## 5. Implementation Examples

### 5.1 Prepare the development environment

```bash
python -m venv .venv
source .venv/bin/activate

pip install \
  boto3 \
  sagemaker \
  pandas \
  scikit-learn \
  xgboost
```

Use a compatible package set for the selected SageMaker training image and runtime.

### 5.2 Upload a training dataset

```python
import boto3

s3 = boto3.client("s3")

s3.upload_file(
    "equipment_training.csv",
    "YOUR_ML_DATA_BUCKET",
    "datasets/equipment-v1/training.csv",
)
```

Use IAM roles rather than embedding AWS credentials in code.

### 5.3 Configure SageMaker training

The following example demonstrates the SageMaker SDK estimator pattern. It assumes a compatible training script and input data already exist.

```python
import sagemaker
from sagemaker.estimator import Estimator

session = sagemaker.Session()

role = sagemaker.get_execution_role()

estimator = Estimator(
    image_uri="YOUR_TRAINING_IMAGE_URI",
    role=role,
    instance_count=1,
    instance_type="ml.m5.large",
    output_path="s3://YOUR_ML_DATA_BUCKET/models/",
    sagemaker_session=session,
    hyperparameters={
        "max_depth": 5,
        "learning_rate": 0.05,
        "n_estimators": 200,
    },
)

estimator.fit({
    "training": "s3://YOUR_ML_DATA_BUCKET/datasets/equipment-v1/"
})
```

Replace placeholders with valid resources in your own AWS account. The training image must support the selected algorithm and hyperparameters.

### 5.4 Deploy a trained model

After successful training and evaluation, a compatible model can be deployed using the SageMaker SDK.

```python
predictor = estimator.deploy(
    initial_instance_count=1,
    instance_type="ml.m5.large",
)
```

This creates a managed real-time endpoint. In production, deployment should be controlled through an approved model version and a release pipeline.

### 5.5 Invoke an endpoint from Lambda

Example Lambda inference handler:

```python
import json
import boto3
import os

runtime = boto3.client("sagemaker-runtime")

ENDPOINT_NAME = os.environ["MODEL_ENDPOINT_NAME"]


def lambda_handler(event, context):
    payload = json.loads(event["body"])

    response = runtime.invoke_endpoint(
        EndpointName=ENDPOINT_NAME,
        ContentType="application/json",
        Body=json.dumps(payload),
    )

    prediction = response["Body"].read().decode("utf-8")

    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json"
        },
        "body": prediction,
    }
```

Configure the Lambda execution role with permission to invoke only the intended endpoint. Validate input types and ranges before invoking the model.

### 5.6 Expose inference through API Gateway

A typical request flow:

```text
Client Application
       |
       v
Amazon API Gateway
       |
       v
AWS Lambda
       |
       v
SageMaker Endpoint
       |
       v
Prediction Response
```

Configure authentication, request validation, throttling, logging, and appropriate API error responses.

## 6. Security and Reliability

### Identity and access management

- Use dedicated execution roles for SageMaker, Lambda, and training jobs.
- Grant access only to required S3 prefixes and model resources.
- Restrict endpoint invocation permissions.
- Avoid hard-coded access keys.
- Use separate development and production environments.

### Data protection

- Encrypt S3 datasets and model artifacts.
- Enable encryption for supported SageMaker resources.
- Use TLS for API communication.
- Store secrets in AWS Secrets Manager when required.
- Apply S3 bucket policies and public access blocks.

### Reliability

- Use endpoint health checks and monitoring.
- Set API Gateway throttling limits.
- Implement bounded retries with exponential backoff.
- Define model rollback procedures.
- Keep previous approved model versions available.
- Use infrastructure as code for repeatable environments.

### Model governance

Track:

- Training dataset version.
- Code revision.
- Hyperparameters.
- Evaluation metrics.
- Model artifact location.
- Approval status.
- Deployment configuration.
- Production model version.

This provides traceability and supports controlled model releases.

## 7. Testing and Troubleshooting

### Model testing

Evaluate models using metrics appropriate to the problem:

| Metric | Purpose |
|---|---|
| Precision | Measures the proportion of positive predictions that are correct |
| Recall | Measures how many actual positive cases are detected |
| F1-score | Balances precision and recall |
| ROC-AUC | Measures ranking performance across thresholds |
| Confusion matrix | Shows true and false predictions |

For predictive maintenance, missed failures can be expensive. Select decision thresholds based on business risk rather than optimizing accuracy alone.

### API testing

Example:

```bash
curl -X POST "YOUR_API_ENDPOINT" \
  -H "Content-Type: application/json" \
  -d '{
    "equipment_id": "machine-104",
    "temperature": 72.4,
    "vibration": 0.83,
    "operating_hours": 1820,
    "pressure": 4.7
  }'
```

### Operational troubleshooting

| Problem | Investigation |
|---|---|
| Training job fails | Review SageMaker training logs and execution role |
| Container image unavailable | Check ECR repository, image URI, and permissions |
| Endpoint creation fails | Check model artifact, instance capacity, and configuration |
| Lambda cannot invoke model | Verify endpoint name and `InvokeEndpoint` permission |
| API returns 5xx | Inspect API Gateway integration and Lambda logs |
| High inference latency | Review endpoint instance type, payload size, and concurrency |
| Model quality declines | Investigate data drift, concept drift, and feature changes |

Use CloudWatch Logs and metrics to correlate failures across the API, Lambda, and SageMaker endpoint.

## 8. Cost and Performance Optimization

### Training

- Use smaller instances for development experiments.
- Run training jobs only when needed.
- Track experiment duration and resource utilization.
- Use managed Spot Training when workloads can tolerate interruptions.

### Inference

- Select endpoint instance types based on measured latency and throughput.
- Configure autoscaling based on invocation load.
- Consider serverless inference for suitable intermittent workloads.
- Evaluate asynchronous inference for large or long-running requests.
- Delete unused endpoints.

### Storage

- Apply S3 lifecycle policies to old datasets and artifacts.
- Retain only required model versions.
- Avoid unnecessary duplication of training datasets.
- Use compression and suitable data formats.

### Governance

- Monitor service spending with AWS Cost Explorer.
- Configure AWS Budgets.
- Track cost per training run and per inference request.
- Review unused resources regularly.

## 9. Official AWS Documentation

- [Amazon SageMaker AI Documentation](https://docs.aws.amazon.com/sagemaker/)
- [SageMaker Pipelines](https://docs.aws.amazon.com/sagemaker/latest/dg/pipelines.html)
- [SageMaker Model Registry](https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html)
- [SageMaker Model Monitor](https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor.html)
- [Amazon ECR Documentation](https://docs.aws.amazon.com/ecr/)
- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [Amazon API Gateway Documentation](https://docs.aws.amazon.com/apigateway/)
- [Amazon CloudWatch Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
- [AWS Well-Architected Machine Learning Lens](https://docs.aws.amazon.com/wellarchitected/latest/machine-learning-lens/welcome.html)

## 10. Conclusion

An effective MLOps platform standardizes the path from experimentation to production.

This predictive maintenance reference project demonstrates how AWS services can support:

1. Reproducible model development.
2. Automated training and evaluation.
3. Versioned model registration.
4. Controlled deployment and inference.
5. Operational monitoring.
6. Continuous model improvement.

The same engineering principles apply to fraud detection, demand forecasting, predictive maintenance, customer analytics, and other production ML workloads.

---

## Publication and Attribution

This tutorial is independently authored for educational purposes. The use case, examples, data fields, and architecture are original. AWS service names and official documentation remain the property of their respective owners.

The examples are educational starting points, not a complete production deployment. Validate SDK compatibility, IAM permissions, model behavior, service limits, and costs before deploying workloads.

This tutorial does not claim guaranteed copyright compliance and does not reproduce the certification assessment.

**Disclaimer:** This is an independently designed educational reference architecture, not a production deployment blueprint or a guarantee of copyright or licensing compliance.
