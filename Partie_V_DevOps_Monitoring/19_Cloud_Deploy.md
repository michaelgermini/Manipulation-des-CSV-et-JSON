# Chapitre 19 : Cloud Deploy — AWS, GCP, Azure

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Déployer** sur **AWS** (Lambda, ECS, S3)
- **Configurer** **GCP** (Cloud Functions, Dataflow)
- **Utiliser** **Azure** (Functions, Blob Storage)
- **Optimiser** coûts cloud
- **Migrer** entre clouds

---

## 📖 Table des matières

1. [AWS Ecosystem](#aws-ecosystem)
2. [Google Cloud Platform](#google-cloud-platform)
3. [Microsoft Azure](#microsoft-azure)
4. [Comparaison & Costs](#comparaison--costs)
5. [Migration & Portabilité](#migration--portabilité)

---

## AWS Ecosystem

### Lambda pour traitement CSV

```python
# lambda_function.py
import json
import boto3
import pandas as pd
from io import BytesIO

s3 = boto3.client('s3')

def lambda_handler(event, context):
    """Process CSV from S3, return JSON"""
    
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Lire CSV from S3
    obj = s3.get_object(Bucket=bucket, Key=key)
    df = pd.read_csv(obj['Body'])
    
    # Traiter
    df_clean = df.dropna()
    df_clean['new_col'] = df_clean['a'] + df_clean['b']
    
    # Sauvegarder JSON
    output = df_clean.to_json(orient='records')
    
    s3.put_object(
        Bucket=bucket,
        Key=key.replace('.csv', '.json'),
        Body=output
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'Processed {len(df_clean)} rows')
    }
```

### S3 + Lambda Event-driven

```bash
# Créer bucket S3
aws s3 mb s3://csv-processor-bucket

# Deploy Lambda
zip -r lambda.zip lambda_function.py
aws lambda create-function \
  --function-name csv-processor \
  --runtime python3.11 \
  --role arn:aws:iam::ACCOUNT:role/lambda-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda.zip \
  --timeout 300 \
  --memory-size 3008

# Add S3 trigger
aws s3api put-bucket-notification-configuration \
  --bucket csv-processor-bucket \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [{
      "LambdaFunctionArn": "arn:aws:lambda:region:account:function:csv-processor",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {"Key": {"FilterRules": [{"Name": "suffix", "Value": ".csv"}]}}
    }]
  }'
```

### ECS for continuous processing

```hcl
# main.tf
resource "aws_ecs_cluster" "csv_cluster" {
  name = "csv-processor"
}

resource "aws_ecs_task_definition" "csv_task" {
  family                   = "csv-processor"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"

  container_definitions = jsonencode([
    {
      name  = "csv-processor"
      image = "${aws_ecr_repository.csv_repo.repository_url}:latest"
      portMappings = [{
        containerPort = 8000
        protocol      = "tcp"
      }]
      environment = [
        { name = "S3_BUCKET", value = aws_s3_bucket.csv_bucket.id },
        { name = "OUTPUT_BUCKET", value = aws_s3_bucket.output_bucket.id }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.csv_logs.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = "ecs"
        }
      }
    }
  ])
}

resource "aws_ecs_service" "csv_service" {
  name            = "csv-processor-service"
  cluster         = aws_ecs_cluster.csv_cluster.id
  task_definition = aws_ecs_task_definition.csv_task.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = aws_subnet.private[*].id
    security_groups = [aws_security_group.ecs.id]
  }
}
```

---

## Google Cloud Platform

### Cloud Functions for JSON

```python
# main.py
import functions_framework
from google.cloud import storage
import json
import pandas as pd
from io import StringIO

storage_client = storage.Client()

@functions_framework.cloud_event
def process_json_gcs(cloud_event):
    """Process JSON from GCS"""
    
    file_name = cloud_event['name']
    bucket_name = cloud_event['bucket']
    
    # Lire JSON
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(file_name)
    json_str = blob.download_as_string()
    
    # Parse et process
    data = json.loads(json_str)
    df = pd.DataFrame(data)
    
    # Nettoyer
    df_clean = df.dropna()
    
    # Export CSV
    csv_buffer = StringIO()
    df_clean.to_csv(csv_buffer, index=False)
    
    output_blob = bucket.blob(file_name.replace('.json', '.csv'))
    output_blob.upload_from_string(csv_buffer.getvalue())
    
    print(f"Processed {len(df_clean)} rows")
```

### Dataflow (Beam) for big data

```python
# dataflow_pipeline.py
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
import json

def parse_csv_line(line):
    if isinstance(line, bytes):
        line = line.decode('utf-8')
    parts = line.strip().split(',')
    return {
        'id': int(parts[0]),
        'name': parts[1],
        'email': parts[2]
    }

def filter_valid(record):
    return '@' in record['email']

def format_output(record):
    return json.dumps(record)

options = PipelineOptions(
    runner='DataflowRunner',
    project='my-project',
    region='us-central1',
    staging_location='gs://my-bucket/staging',
    temp_location='gs://my-bucket/temp',
    save_main_session=True
)

with beam.Pipeline(options=options) as pipeline:
    (pipeline
     | 'Read' >> beam.io.ReadFromText('gs://my-bucket/input.csv')
     | 'Parse' >> beam.Map(parse_csv_line)
     | 'Filter' >> beam.Filter(filter_valid)
     | 'Format' >> beam.Map(format_output)
     | 'Write' >> beam.io.WriteToText('gs://my-bucket/output'))
```

---

## Microsoft Azure

### Azure Functions with Blob Storage

```python
# __init__.py
import azure.functions as func
from azure.storage.blob import BlobClient
import pandas as pd
import json

def main(myblob: func.InputStream, context: func.Context) -> None:
    """Process CSV from Azure Blob Storage"""
    
    # Lire blob
    csv_content = myblob.read().decode('utf-8')
    
    # Parse CSV
    from io import StringIO
    df = pd.read_csv(StringIO(csv_content))
    
    # Process
    df_clean = df.dropna()
    df_clean['processed_at'] = pd.Timestamp.now()
    
    # Upload résultat
    connection_string = os.environ['AzureWebJobsStorage']
    blob_client = BlobClient.from_connection_string(
        connection_string,
        container_name='outputs',
        blob_name=f"{myblob.name.replace('.csv', '.json')}"
    )
    
    json_output = df_clean.to_json(orient='records')
    blob_client.upload_blob(json_output, overwrite=True)
    
    context.log(f"Processed {len(df_clean)} records")
```

### Azure Data Factory Pipeline

```json
{
  "name": "CSV2JSONPipeline",
  "properties": {
    "activities": [
      {
        "name": "CopyCSV",
        "type": "Copy",
        "inputs": [{
          "referenceName": "CSVDataset",
          "type": "DatasetReference"
        }],
        "outputs": [{
          "referenceName": "JSONDataset",
          "type": "DatasetReference"
        }],
        "typeProperties": {
          "source": {
            "type": "DelimitedTextSource"
          },
          "sink": {
            "type": "JsonSink"
          }
        }
      },
      {
        "name": "ProcessJSON",
        "type": "AzureFunction",
        "dependsOn": [{
          "activity": "CopyCSV",
          "dependencyConditions": ["Succeeded"]
        }],
        "typeProperties": {
          "functionAppUrl": "https://myapp.azurewebsites.net",
          "functionName": "ProcessData",
          "authentication": "Anonymous"
        }
      }
    ]
  }
}
```

---

## Comparaison & Costs

### Pricing Comparison (100GB CSV processing)

```
Use case: Process 100GB CSV monthly, 1000 files

AWS:
├─ Lambda: $0.0000002 * 100GB * 300s = $60
├─ S3: 100GB × $0.023 = $2,300
├─ Bandwidth: 100GB × $0.09 = $9,000
└─ Total: ~$11,360/month

GCP:
├─ Cloud Functions: $0.0000004 * invocations = $15
├─ GCS: 100GB × $0.020 = $2,000
├─ Bandwidth: 100GB × $0.12 = $12,000
└─ Total: ~$14,015/month

Azure:
├─ Functions: $0.20 per 1M executions = $10
├─ Blob: 100GB × $0.0184 = $1,840
├─ Bandwidth: 100GB × $0.087 = $8,700
└─ Total: ~$10,550/month (cheapest)

Winner: Azure (~7% cheaper than AWS)
```

### Decision Matrix

| Criteria | AWS | GCP | Azure |
|----------|-----|-----|-------|
| **Ease of use** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **CSV/JSON tools** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Pricing** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Performance** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Ecosystem** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |

---

## Migration & Portabilité

### Containerize for multi-cloud

```dockerfile
FROM python:3.11

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["python", "processor.py"]
```

### Deploy anywhere with Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: csv-processor
spec:
  replicas: 3
  selector:
    matchLabels:
      app: csv-processor
  template:
    metadata:
      labels:
        app: csv-processor
    spec:
      containers:
      - name: processor
        image: myrepo/csv-processor:latest
        env:
        - name: INPUT_BUCKET
          value: gs://my-bucket/input
        - name: OUTPUT_BUCKET
          value: gs://my-bucket/output
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
```

Deploy to any cloud:
```bash
# AWS EKS
eksctl create cluster --name csv-processor
kubectl apply -f deployment.yaml

# GCP GKE
gcloud container clusters create csv-processor
kubectl apply -f deployment.yaml

# Azure AKS
az aks create --name csv-processor
kubectl apply -f deployment.yaml
```

---

## 🎓 Exercices pratiques

### Exercice 19.1 : Lambda
Créez Lambda AWS pour traiter CSV.

### Exercice 19.2 : Dataflow
Pipeline GCP Dataflow pour JSON.

### Exercice 19.3 : Azure Functions
Azure Function pour transformation.

### Exercice 19.4 : Costs
Comparez pricing 3 clouds.

### Exercice 19.5 : Migration
Migrez solution AWS → GCP.

---

## 📚 Références

- **AWS Lambda** : https://aws.amazon.com/lambda/
- **GCP Dataflow** : https://cloud.google.com/dataflow
- **Azure Functions** : https://azure.microsoft.com/services/functions/

---

**Prêt pour le backup? → [Chapitre 22 : Backup & DR](./22_Backup_DR.md)**
