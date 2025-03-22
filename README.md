# RDB_TO_VDB

ETL Pipeline: From Relational Data Base To Vector Data Base

## Source DB

Relational Database: PostgreSQL

## Target DB

Vector Database: Elasticsearch

## 1. Directory Schema

```
RDB_TO_VDB/
├── .env
├── docker-compose.yml
├── airflow/
│   ├── dags/
│   │   └── etl_dag.py
│   ├── plugins/
│   └── Dockerfile # Airflow 커스텀 이미지 추가
├── src/
│   ├── etl/
│   │   ├── extract.py
│   │   ├── transform.py
│   │   └── load.py
│   ├── monitoring/
│   │   └── metrics.py
│   └── config/
│       ├── __init__.py
│       ├── settings.py # .env 로드 기능 추가
│       └── constants.py
├── tests/
│   ├── test_extract.py
│   ├── test_transform.py
│   └── test_load.py
└── prometheus/
    ├── prometheus.yml
    └── alerts.yml
```

## 2. Environment Setting

```bash
# UV 설치
curl -LsSf https://astral.sh/uv/install.sh | sh

# 프로젝트 초기화
uv venv .venv -p 3.11
source .venv/bin/activate

# 종속성 관리
uv pip install -r airflow/requirements.txt -r src/requirements.txt
```

## 3. ETL pipeline configuration

### 3-1. From Source DB(PostgreSQL) To Target DB(Elasticsearch) Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    networks:
      - etl-net
    image: postgres:15
    environment:
      POSTGRES_USER: etl_user
      POSTGRES_PASSWORD: etl_pass
      POSTGRES_DB: source_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U etl_user -d source_db"]
      interval: 5s
      timeout: 5s
      retries: 5

  elasticsearch:
    networks:
      - etl-net
    image: elasticsearch:8.11.4
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - ELASTIC_PASSWORD=elastic_pass
      - xpack.security.enabled=true
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - elastic_data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -u elastic:elastic_pass -f http://elasticsearch:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  init-elastic:
    networks:
      - etl-net
    image: appropriate/curl:latest
    depends_on:
      elasticsearch:
        condition: service_healthy
    command: >
       sh -c '
       until curl -u elastic:elastic_pass -X PUT "http://elasticsearch:9200/documents"
         -H "Content-Type: application/json"
         -d "{\"mappings\":{\"properties\":{\"content\":{\"type\":\"text\"},\"vector_data\":{\"type\":\"dense_vector\",\"dims\":3,\"index\":true,\"similarity\":\"cosine\"}}}}";
       do sleep 10; done'

  airflow:
    networks:
      - etl-net
    build: ./airflow
    ports:
      - "8081:8080"
    environment:
      - AIRFLOW__CORE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres:5432/airflow
    depends_on:
      postgres:
        condition: service_healthy

  prometheus:
    networks:
      - etl-net
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  etl:
    networks:
      - etl-net
    build: ./src
    dockerfile: Dockerfile
    container_name: rdb_to_vdb_etl
    environment:
      - PYTHONPATH=/app/src
    ports:
      - "9100:9100"
    depends_on:
      postgres:
        condition: service_healthy
      elasticsearch:
        condition: service_healthy

volumes:
  postgres_data:
  elastic_data:

networks:
  etl-net:
    driver: bridge
```

### 3-2. Airflow Configuration

```Dockerfile
# airflow/Dockerfile
FROM apache/airflow:2.7.2-python3.11
USER root
RUN apt-get update && \
    apt-get install -y gcc python3-dev && \
    apt-get clean
USER airflow
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

## 4. ETL 파이프라인 (RDB TO VDB)

### 4-1. 환경 변수 설정 (.env)

```
# PostgreSQL
PG_HOST=postgres
PG_PORT=5432
PG_DB=source_db
PG_USER=etl_user
PG_PASSWORD=etl_pass

# Elasticsearch
ES_HOST=elasticsearch
ES_PORT=9200
ES_INDEX=documents
ES_USER=elastic
ES_PASSWORD=elastic_pass

# PATH
PYTHONPATH="${PYTHONPATH}:/app/src"
AIRFLOW_HOME=/app/airflow

# 모니터링
METRICS_PORT=9100
```

### 4-2. 메트릭 수집 예시

```python
# src/monitoring/metrics.py
import os
from prometheus_client import start_http_server, Counter, Gauge, Histogram

PROCESSED_DOCS = Counter('etl_processed_documents', 'Total processed documents')
PROCESSING_TIME = Histogram('etl_processing_time', 'Document processing time')
FAILED_DOCS = Counter('etl_failed_documents', 'Total failed documents')
BATCH_SIZE = Gauge('etl_batch_size', 'Current processing batch size')

def start_monitoring():
    start_http_server(int(os.getenv('METRICS_PORT', 9100)))
```

## 5. Scheduling(Batch): Airflow DAG 구성

### 5-1. ETL DAG Example

```python
# airflow/dags/etl_dag.py
from datetime import datetime
from datetime import timedelta
from airflow.utils.dates import days_ago
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.models import Variable
from src.etl import run_etl_pipeline

default_args = {
    'owner': 'etl',
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'depends_on_past': False
}

with DAG(
    'document_etl',
    default_args=default_args,
    schedule_interval='@hourly',
    start_date=days_ago(1),
    catchup=False
) as dag:
    etl_task = PythonOperator(
        task_id='run_etl',
        python_callable=run_etl_pipeline,
        execution_timeout=timedelta(minutes=45),
    )
```

## 6. Connect Open Source Monitoring: Prometheus

### 6-1. Prometheus Setting

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'etl_metrics'
    static_configs:
      - targets: ['rdb_to_vdb_etl:9100']

  - job_name: 'airflow'
    metrics_path: '/admin/metrics/'
    static_configs:
      - targets: ['airflow:8080']
```

### 6-2. Grafana Dashboard Setting

```
- Prometheus 데이터소스 추가
1. Grafana에서 "Configuration" > "Data Sources" 선택
2. Prometheus 타입 선택 후 URL 입력 (http://prometheus:9090)
3. Save & Test 클릭
```

### 6-3. Sample Dashboard Configration

```
{
  "title": "ETL Pipeline Metrics",
  "editable": true,
   "panels": [
     {
       "type": "graph",
       "title": "문서 처리량",
      "gridPos": {"x":0,"y":0,"w":12,"h":8},
       "targets": [{
         "expr": "rate(etl_processed_documents_total[1h])",
         "legendFormat": "처리량"
       }]
     },
     {
       "type": "stat",
       "title": "실패율",
       "gridPos": {"x":12,"y":0,"w":12,"h":8},
       "targets": [{
         "expr": "etl_failed_documents_total / etl_processed_documents_total",
         "format": "percent"
       }]
     }
   ]
}
```

## 7. ETL Workflow

```
sequenceDiagram
    participant Airflow
    participant PostgreSQL
    participant ETL
    participant Elasticsearch
    participant Prometheus
    participant Grafana
    
    Airflow->>ETL: HTTP 요청 전송
    ETL->>PostgreSQL: 배치 데이터 추출 (500건 단위)
    PostgreSQL->>ETL: 데이터 반환
    ETL->>Elasticsearch: Basic Auth 인증 포함
    ETL->>Prometheus: 처리 메트릭 전송
    Prometheus->>Grafana: 데이터 수집
    Grafana-->>개발자: 실시간 모니터링 + 이상치 경고
```

### 7-1. ETL Pipeline Execute Test

```bash
# 전체 시스템 시작
docker-compose up -d postgres elasticsearch airflow prometheus

# Airflow DB 초기화
docker-compose exec airflow airflow db init

# 테스트 데이터 생성 (PostgreSQL)
docker-compose exec postgres psql -U etl_user -d source_db -c "INSERT INTO documents (content, vector_data) VALUES ('테스트', '{0.1,0.2,0.3}');"

# 파이프라인 수동 실행
docker-compose exec airflow airflow dags trigger document_etl
```