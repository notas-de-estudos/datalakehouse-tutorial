# 📘 Apache Iceberg

## 📌 Introdução
Este tutorial apresenta os conceitos fundamentais do **Data Lakehouse**, com foco no framework **Apache Iceberg**, um formato de tabela moderno que unifica processamento analítico em data lakes com governança, consistência e performance.

O objetivo é mostrar **como Iceberg funciona**, quais problemas resolve e como utilizá-lo em conjunto com serviços da AWS e motores de processamento como Spark.

---

## 🧊 O que é o Apache Iceberg?
O **Apache Iceberg** é um *open table format* projetado para armazenar e gerenciar tabelas analíticas diretamente sobre armazenamentos como S3, HDFS e GCS.

Ele fornece:

- Transações ACID-like  
- Snapshots versionados  
- Time Travel  
- Schema Evolution sem quebra  
- Suporte multi-engine (Spark, Flink, Trino/Presto, Hive)

Iceberg traz para o data lake funcionalidades semelhantes a bancos analíticos modernos, mas mantendo custos menores e sem lock-in.

---

## 🎬 Case Real: Netflix
A Netflix adotou Iceberg para escalar suas operações de dados:

- Mais de **10 PB** de dados analíticos gerenciados por Iceberg  
- Migração de **1,5 milhão de tabelas Hive** para Iceberg  
- Necessidade de interoperabilidade:  
  - Cientistas usando **Spark**  
  - Analistas e BI usando **Presto/Trino**

Iceberg se destacou pela performance, governança e facilidade de manutenção.

---

## 🏛 Arquitetura do Iceberg

### ✔ Metadata Layer
Iceberg separa **metadados** dos arquivos físicos. Isso permite:

- Snapshots versionados  
- Manifestos para indexar arquivos  
- Particionamento evolutivo  
- Evitar full scans desnecessários

### Componentes principais:

| Componente | Função |
|-----------|--------|
| **Snapshot** | Representa o estado completo da tabela em um momento |
| **Manifest List** | Lista de manifestos pertencentes a um snapshot |
| **Manifest File** | Índice de arquivos (Parquet/ORC), estatísticas e partições |
| **Data Files** | Arquivos físicos de dados (Parquet/ORC) |
| **Delete Files** | Arquivos contendo operações de delete |
| **Catalog** | Local onde a tabela é registrada (Glue, Hive Metastore, Nessie, REST) |

![Arquitetura Apache Iceberg](/2-apache-iceberg/img/arquitetura-iceberg.png)

![Arquitetura Iceberg e Glue](/2-apache-iceberg/img/iceberg-glue.png)

---

## ⭐ Vantagens e Recursos-Chave

### 🔹 Time Travel & Snapshots
Permite consultar versões antigas da tabela ou fazer rollback.

### 🔹 Schema Evolution Seguro
Colunas possuem IDs; mudanças como rename/drop não quebram queries.

### 🔹 Hidden Partitioning
O usuário consulta como tabela SQL comum, sem expor estrutura de partições.

### 🔹 Isolation e Consistência
Leitores sempre veem um snapshot consistente, mesmo durante escrituras.

---

## Exemplo de Uso com AWS Glue, S3 e Spark (SQL, e PySpark)

### 1. Criar catálogo Iceberg usando o AWS Glue

```python
spark = (
    SparkSession.builder
        .appName("IcebergGlue")
        .config("spark.sql.catalog.glue", "org.apache.iceberg.spark.SparkCatalog")
        .config("spark.sql.catalog.glue.catalog-impl", "org.apache.iceberg.aws.glue.GlueCatalog")
        .config("spark.sql.catalog.glue.warehouse", "s3://meu-bucket/warehouse/")
        .config("spark.sql.catalog.glue.io-impl", "org.apache.iceberg.aws.s3.S3FileIO")
        .getOrCreate()
)
```

### 2. Criar tabela Iceberg no S3

```sql
CREATE TABLE glue.db.pedidos (
  id BIGINT,
  cliente STRING,
  valor DOUBLE,
  ts TIMESTAMP
)
USING iceberg
PARTITIONED BY (days(ts));
```

### 3. Inserir dados via PySpark

```python
df = spark.createDataFrame([
    (1, "João", 150.0, "2025-01-01 10:00:00")
], ["id", "cliente", "valor", "ts"])

df.write.format("iceberg").mode("append").save("glue.db.pedidos")
```

### 4. Consultar tabela e snapshots

- Leitura normal

```python
spark.read.format("iceberg").load("glue.db.pedidos").show()
```

- Time Travel por timestamp

```SQL
SELECT * FROM glue.db.pedidos
FOR SYSTEM_TIME AS OF TIMESTAMP '2025-01-01 10:05:00';
```