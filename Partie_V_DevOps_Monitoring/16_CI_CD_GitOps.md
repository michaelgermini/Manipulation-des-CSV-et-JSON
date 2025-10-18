# Chapitre 16 : CI/CD & GitOps — Automation et Déploiement

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Automatiser tests et déploiements
- **GitHub Actions**, **GitLab CI**, **Jenkins**
- Containeriser pipelines (Docker)
- Infrastructure-as-Code (Terraform)
- Rollouts progressifs (Canary, Blue-Green)

---

## 📖 Table des matières

1. [GitHub Actions](#github-actions)
2. [GitLab CI/CD](#gitlab-cicd)
3. [Docker & Containerization](#docker--containerization)
4. [Infrastructure-as-Code](#infrastructure-as-code)
5. [Deployment strategies](#deployment-strategies)
6. [Monitoring et rollback](#monitoring-et-rollback)

---

## GitHub Actions

### Simple workflow

```yaml
# .github/workflows/csv-pipeline.yml
name: CSV ETL Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install pytest pytest-cov
    
    - name: Run tests
      run: pytest tests/ --cov=src --cov-report=xml
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage.xml

  lint:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Lint with ruff
      run: |
        pip install ruff
        ruff check .
        ruff format --check .
    
    - name: Type check with mypy
      run: |
        pip install mypy
        mypy src/

  build-and-deploy:
    needs: [test, lint]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: myrepo/csv-pipeline:${{ github.sha }}
        tags: myrepo/csv-pipeline:latest
    
    - name: Deploy to staging
      run: |
        echo "Deploying to staging..."
        # kubectl apply -f k8s/staging/
```

### Test workflow

```yaml
# .github/workflows/test.yml
name: Test CSV Processing

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        python-version: ['3.9', '3.10', '3.11']
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    
    - name: Test with pytest
      run: |
        pytest tests/ -v --tb=short
```

---

## GitLab CI/CD

### Full CI/CD pipeline

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - staging
  - production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""

build-image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

test-unit:
  stage: test
  image: python:3.11
  script:
    - pip install -r requirements.txt pytest pytest-cov
    - pytest tests/ --cov=src --cov-report=term --cov-report=html
  artifacts:
    paths:
      - htmlcov/
    expire_in: 30 days
  coverage: '/TOTAL.*\s+(\d+%)$/'

test-integration:
  stage: test
  image: python:3.11
  services:
    - postgres:13
    - elasticsearch:7.17.0
  variables:
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
  script:
    - pip install -r requirements.txt pytest
    - pytest tests/integration/ -v
  only:
    - main

deploy-staging:
  stage: staging
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/csv-pipeline csv-pipeline=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/csv-pipeline
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main

deploy-production:
  stage: production
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/csv-pipeline csv-pipeline=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/csv-pipeline
  environment:
    name: production
    url: https://example.com
  when: manual  # Manual approval required
  only:
    - main
```

---

## Docker & Containerization

### Dockerfile for CSV pipeline

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY src/ ./src/
COPY data/ ./data/
COPY config/ ./config/

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import sys; sys.exit(0)"

# Run pipeline
CMD ["python", "-m", "src.pipeline"]
```

### Docker Compose for full stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  csv-pipeline:
    build: .
    environment:
      - ELASTICSEARCH_HOST=elasticsearch:9200
      - POSTGRES_HOST=postgres
      - POSTGRES_USER=csvuser
      - POSTGRES_PASSWORD=secret
    depends_on:
      - elasticsearch
      - postgres
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    ports:
      - "8000:8000"

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: csvuser
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: pipeline_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

volumes:
  es_data:
  postgres_data:
```

---

## Infrastructure-as-Code

### Terraform for AWS

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# ECS Cluster
resource "aws_ecs_cluster" "csv_pipeline" {
  name = "csv-pipeline-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# Task Definition
resource "aws_ecs_task_definition" "csv_pipeline" {
  family                   = "csv-pipeline"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"

  container_definitions = jsonencode([
    {
      name      = "csv-pipeline"
      image     = "${aws_ecr_repository.csv_pipeline.repository_url}:latest"
      essential = true
      portMappings = [
        {
          containerPort = 8000
          hostPort      = 8000
          protocol      = "tcp"
        }
      ]
      environment = [
        {
          name  = "ELASTICSEARCH_HOST"
          value = aws_opensearch_domain.pipeline.endpoint
        }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.pipeline.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = "ecs"
        }
      }
    }
  ])
}

# Service
resource "aws_ecs_service" "csv_pipeline" {
  name            = "csv-pipeline-service"
  cluster         = aws_ecs_cluster.csv_pipeline.id
  task_definition = aws_ecs_task_definition.csv_pipeline.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs.id]
    assign_public_ip = false
  }

  depends_on = [aws_lb_listener.http]
}

# Auto Scaling
resource "aws_appautoscaling_target" "csv_pipeline" {
  max_capacity       = 10
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.csv_pipeline.name}/${aws_ecs_service.csv_pipeline.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  policy_name            = "cpu-autoscaling"
  policy_type            = "TargetTrackingScaling"
  resource_id            = aws_appautoscaling_target.csv_pipeline.resource_id
  scalable_dimension     = aws_appautoscaling_target.csv_pipeline.scalable_dimension
  service_namespace      = aws_appautoscaling_target.csv_pipeline.service_namespace
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

---

## Deployment strategies

### Blue-Green deployment

```yaml
# k8s/blue-green.yml
apiVersion: v1
kind: Service
metadata:
  name: csv-pipeline
spec:
  selector:
    app: csv-pipeline
  ports:
    - port: 80
      targetPort: 8000
  type: LoadBalancer

---
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: csv-pipeline-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: csv-pipeline
      version: blue
  template:
    metadata:
      labels:
        app: csv-pipeline
        version: blue
    spec:
      containers:
      - name: pipeline
        image: myrepo/csv-pipeline:v1.0.0
        ports:
        - containerPort: 8000

---
# Green deployment (new)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: csv-pipeline-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: csv-pipeline
      version: green
  template:
    metadata:
      labels:
        app: csv-pipeline
        version: green
    spec:
      containers:
      - name: pipeline
        image: myrepo/csv-pipeline:v1.1.0  # New version
        ports:
        - containerPort: 8000
```

### Canary deployment

```yaml
# k8s/canary.yml
apiVersion: fluxcd.io/v1alpha1
kind: Canary
metadata:
  name: csv-pipeline-canary
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: csv-pipeline
  progressDeadlineSeconds: 300
  service:
    port: 80
  analysis:
    interval: 30s
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: error-rate
      thresholdRange:
        max: 0.05  # Error rate < 5%
      interval: 60s
    - name: latency
      thresholdRange:
        max: 500  # P99 < 500ms
      interval: 60s
```

---

## Monitoring et rollback

### Automated rollback

```python
# deploy_with_monitoring.py
import requests
import time
from datetime import datetime

def deploy_with_monitoring(new_image: str, rollback_on_error: bool = True):
    """Deploy with automatic rollback on metrics failure"""
    
    old_image = get_current_image()
    
    # Deploy new version
    print(f"[{datetime.now()}] Deploying {new_image}...")
    deploy_image(new_image)
    
    # Monitor for 5 minutes
    start_time = time.time()
    monitoring_duration = 300  # 5 minutes
    
    while time.time() - start_time < monitoring_duration:
        metrics = get_metrics()
        
        # Check SLOs
        if metrics['error_rate'] > 0.05:  # > 5%
            print(f"❌ High error rate: {metrics['error_rate']:.2%}")
            if rollback_on_error:
                print(f"Rolling back to {old_image}...")
                deploy_image(old_image)
                notify("Automatic rollback executed", severity="HIGH")
                return False
        
        if metrics['p99_latency'] > 1000:  # > 1s
            print(f"❌ High latency: {metrics['p99_latency']:.0f}ms")
            if rollback_on_error:
                deploy_image(old_image)
                notify("Automatic rollback executed", severity="HIGH")
                return False
        
        print(f"✓ Health: {metrics['error_rate']:.2%} errors, "
              f"{metrics['p99_latency']:.0f}ms latency")
        
        time.sleep(30)
    
    print(f"✓ Deployment successful!")
    return True
```

---

## 🎓 Exercices pratiques

### Exercice 16.1 : GitHub Actions
Créez workflow pour tester et builder image Docker.

### Exercice 16.2 : Docker
Créez Dockerfile + docker-compose pour pipeline.

### Exercice 16.3 : Terraform
Déployez pipeline sur AWS ECS avec Terraform.

### Exercice 16.4 : Deployment
Implémentez canary deployment avec monitoring.

### Exercice 16.5 : Rollback
Testez rollback automatique sur erreur.

---

## 📚 Références

- **GitHub Actions** : https://github.com/features/actions
- **GitLab CI** : https://docs.gitlab.com/ee/ci/
- **Docker** : https://www.docker.com/
- **Terraform** : https://www.terraform.io/
- **Flux CD** : https://fluxcd.io/

---

**Prêt pour la sécurité? → [Chapitre 18 : Sécurité & Conformité](./18_Securite_Conformite.md)**
