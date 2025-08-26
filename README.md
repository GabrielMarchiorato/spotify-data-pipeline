# Spotify Data Pipeline 🎶

Pipeline de dados completa utilizando **Airflow + Spark + MinIO + Postgres + Streamlit**.

## Arquitetura

* **Ingestão**: Spotify API → MinIO (bronze)
* **Transformação**: Bronze → Silver (Parquet no MinIO) usando Spark
* **Carga**: Silver → Postgres (DW) usando Spark JDBC
* **Visualização**: Streamlit para dashboards interativos

## Estrutura do Projeto

```
spotify-data-pipeline/
├── .gitignore
├── .env.example
├── docker-compose.yml
├── README.md
├── airflow/
│   ├── requirements.txt
│   └── dags/
│       ├── spotify_pipeline.py
│       └── scripts/
│           └── etl_ingest.py
├── spark/
│   ├── etl_transform.py
│   └── etl_load.py
├── streamlit/
│   ├── app.py
│   └── requirements.txt
└── docker/
    └── postgres/
        └── init.sql
```

## Configuração Inicial

1. Copie `.env.example` para `.env` e preencha com suas credenciais do Spotify API.
2. Ajuste as variáveis de Postgres e MinIO, se necessário.

## Como Rodar

No terminal, dentro da pasta do projeto:

```bash
docker compose up -d
```

### UIs:

* Airflow: [http://localhost:8080](http://localhost:8080) (login: admin/admin)
* MinIO Console: [http://localhost:9001](http://localhost:9001) (login: minio/minio123)
* Spark Master UI: [http://localhost:8081](http://localhost:8081)
* Streamlit: [http://localhost:8501](http://localhost:8501)

## Passo a Passo

### 1. Ingestão (Airflow)

No Airflow, DAG `spotify_pipeline`:

* Rode `check_env` para validar variáveis de ambiente.
* Execute `ingest_spotify_to_minio` para coletar dados das playlists e salvar no MinIO (bronze).

### 2. Transformação (Spark)

Execute dentro do container `spark-master`:

```bash
docker compose exec spark-master bash -lc '
  spark-submit --master spark://spark-master:7077 \
  --packages org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262,org.postgresql:postgresql:42.7.4 \
  --conf spark.hadoop.fs.s3a.endpoint=${MINIO_ENDPOINT#http://} \
  --conf spark.hadoop.fs.s3a.access.key=$MINIO_ROOT_USER \
  --conf spark.hadoop.fs.s3a.secret.key=$MINIO_ROOT_PASSWORD \
  --conf spark.hadoop.fs.s3a.path.style.access=true \
  --conf spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem \
  /opt/spark/apps/etl_transform.py --bucket $MINIO_BUCKET'
```

Isso cria a camada Silver (`tracks`, `artists`, `track_artists`, `audio_features`).

### 3. Carga (Silver → Postgres)

```bash
docker compose exec spark-master bash -lc '
  spark-submit --master spark://spark-master:7077 \
  --packages org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262,org.postgresql:postgresql:42.7.4 \
  --conf spark.hadoop.fs.s3a.endpoint=${MINIO_ENDPOINT#http://} \
  --conf spark.hadoop.fs.s3a.access.key=$MINIO_ROOT_USER \
  --conf spark.hadoop.fs.s3a.secret.key=$MINIO_ROOT_PASSWORD \
  --conf spark.hadoop.fs.s3a.path.style.access=true \
  --conf spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem \
  /opt/spark/apps/etl_load.py --bucket $MINIO_BUCKET --jdbc_url $JDBC_URL --jdbc_user $JDBC_USER --jdbc_password $JDBC_PASSWORD'
```

### 4. Visualização (Streamlit)

Acesse [http://localhost:8501](http://localhost:8501) para explorar os dashboards com contagens, top faixas e análises de popularidade.

## Observações

* Variáveis de ambiente sensíveis **não devem ser commitadas**. Use `.env` local.
* Spark lê dados do MinIO e grava no Postgres via JDBC.
* Airflow é usado para orquestração da ingestão e checagem de variáveis.
* Streamlit consome o Postgres para visualizações interativas.

---
