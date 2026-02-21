# PostgreSQL 到 Elasticsearch 实时数据同步方案

## 任务信息
- **任务ID**: cmltory2p0012j679kxw9yjsm
- **技术栈**: CDC, Debezium, PostgreSQL, Elasticsearch, Kafka
- **目标**: 构建生产级的实时数据同步层

---

## 1. 架构设计

### 1.1 整体架构图

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Application   │────▶│    PostgreSQL   │────▶│   Debezium      │
│    (Writes)     │     │  (Source DB)    │     │   Connector     │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                         │
                                                         │ CDC Events
                                                         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Application   │◀────│  Elasticsearch  │◀────│     Kafka       │
│    (Searches)   │     │   (Search DB)   │     │  (Message Bus)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                              ▲                         │
                              │                         │
                              └─────────────────────────┘
                                    Kafka Connect Sink
```

### 1.2 组件说明

| 组件 | 角色 | 说明 |
|------|------|------|
| PostgreSQL | 数据源 | 主数据库，业务数据写入 |
| Debezium | CDC 捕获 | 监控 PostgreSQL WAL，捕获数据变更 |
| Kafka | 消息队列 | 传输变更事件，解耦生产消费 |
| Kafka Connect | 数据同步 | 将 Kafka 消息同步到 Elasticsearch |
| Elasticsearch | 搜索存储 | 面向搜索优化的数据副本 |

### 1.3 数据流说明

1. **写入阶段**: 应用写入 PostgreSQL
2. **CDC 捕获**: Debezium 读取 PostgreSQL WAL (Write-Ahead Log)
3. **事件发布**: 变更事件以 Avro/JSON 格式发布到 Kafka
4. **数据转换**: Kafka Connect 进行数据格式转换
5. **索引写入**: 数据写入 Elasticsearch
6. **搜索服务**: 应用从 Elasticsearch 读取进行搜索

---

## 2. 环境准备

### 2.1 Docker Compose 环境配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL 主数据库
  postgres:
    image: postgres:15-alpine
    container_name: postgres-cdc
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-app_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-secure_password}
      POSTGRES_DB: ${POSTGRES_DB:-app_database}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    command: >
      postgres
      -c wal_level=logical
      -c max_replication_slots=4
      -c max_wal_senders=4
    networks:
      - cdc-network

  # Zookeeper (Kafka 依赖)
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    networks:
      - cdc-network

  # Kafka 消息队列
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
    networks:
      - cdc-network

  # Kafka Connect (包含 Debezium)
  kafka-connect:
    image: debezium/connect:2.4.0.Final
    container_name: kafka-connect
    depends_on:
      - kafka
      - postgres
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:29092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      STATUS_STORAGE_TOPIC: connect_status
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_INTERNAL_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_INTERNAL_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
    volumes:
      - ./connect/plugins:/kafka/connect/plugins
    networks:
      - cdc-network

  # Elasticsearch 搜索引擎
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - cdc-network

  # Kibana (可视化)
  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - cdc-network

  # Schema Registry (Avro 支持)
  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    container_name: schema-registry
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:29092
    networks:
      - cdc-network

  # 监控: Prometheus
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - cdc-network

  # 监控: Grafana
  grafana:
    image: grafana/grafana:10.1.0
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - cdc-network

volumes:
  postgres_data:
  elasticsearch_data:
  prometheus_data:
  grafana_data:

networks:
  cdc-network:
    driver: bridge
```

### 2.2 PostgreSQL 初始化脚本

```sql
-- postgres/init.sql
-- 启用逻辑复制

-- 创建复制用户
CREATE USER replication_user WITH REPLICATION LOGIN PASSWORD 'replication_pass';

-- 创建示例数据库表
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    full_name VARCHAR(200),
    avatar_url TEXT,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    category VARCHAR(100),
    tags TEXT[],
    inventory_count INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    status VARCHAR(50) DEFAULT 'pending',
    total_amount DECIMAL(12, 2),
    shipping_address JSONB,
    items JSONB DEFAULT '[]',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 创建更新触发器函数
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- 为所有表添加自动更新时间戳
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_products_updated_at BEFORE UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_orders_updated_at BEFORE UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- 创建发布以支持逻辑复制
CREATE PUBLICATION dbz_publication FOR TABLE users, products, orders;

-- 授予权限
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replication_user;
GRANT USAGE ON SCHEMA public TO replication_user;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO replication_user;

-- 插入示例数据
INSERT INTO users (username, email, full_name) VALUES
    ('john_doe', 'john@example.com', 'John Doe'),
    ('jane_smith', 'jane@example.com', 'Jane Smith'),
    ('bob_wilson', 'bob@example.com', 'Bob Wilson');

INSERT INTO products (name, description, price, category, tags, inventory_count) VALUES
    ('Wireless Headphones', 'Premium wireless headphones', 199.99, 'Electronics', ARRAY['audio', 'wireless', 'bluetooth'], 100),
    ('Smart Watch', 'Fitness tracking smartwatch', 299.99, 'Electronics', ARRAY['wearable', 'fitness', 'smart'], 50),
    ('Running Shoes', 'Professional running shoes', 129.99, 'Sports', ARRAY['running', 'shoes', 'sport'], 200);
```

---

## 3. Debezium 连接器配置

### 3.1 PostgreSQL Source Connector

```json
{
  "name": "postgres-source-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "replication_user",
    "database.password": "replication_pass",
    "database.dbname": "app_database",
    "database.server.name": "postgres-server",
    "table.include.list": "public.users,public.products,public.orders",
    "plugin.name": "pgoutput",
    "publication.name": "dbz_publication",
    "slot.name": "debezium_slot",
    "snapshot.mode": "initial",
    "decimal.handling.mode": "string",
    "time.precision.mode": "connect",
    "tombstones.on.delete": "true",
    "transforms": "unwrap,router",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "transforms.unwrap.add.fields": "op,ts_ms,source.ts_ms",
    "transforms.router.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.router.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.router.replacement": "$3",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "errors.tolerance": "all",
    "errors.log.enable": "true",
    "errors.log.include.messages": "true",
    "errors.deadletterqueue.topic.name": "dlq-postgres-source",
    "errors.deadletterqueue.context.headers.enable": "true"
  }
}
```

### 3.2 创建连接器脚本

```bash
#!/bin/bash
# scripts/setup-debezium.sh

set -e

CONNECT_URL="http://localhost:8083/connectors"

echo "Waiting for Kafka Connect to be ready..."
until curl -s "${CONNECT_URL}" > /dev/null 2>&1; do
    echo "Waiting for Kafka Connect..."
    sleep 5
done

echo "Creating PostgreSQL source connector..."

curl -X POST "${CONNECT_URL}" \
  -H "Content-Type: application/json" \
  -d @config/postgres-source-connector.json

echo "Source connector created successfully!"

# 验证连接器状态
echo "Checking connector status..."
curl -s "${CONNECT_URL}/postgres-source-connector/status" | jq .
```

---

## 4. Elasticsearch Sink 连接器

### 4.1 Elasticsearch Sink 配置

```json
{
  "name": "elasticsearch-sink-connector",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "3",
    "topics": "users,products,orders",
    "connection.url": "http://elasticsearch:9200",
    "connection.username": "",
    "connection.password": "",
    "type.name": "_doc",
    "key.ignore": "false",
    "schema.ignore": "true",
    "behavior.on.malformed.documents": "warn",
    "behavior.on.null.values": "delete",
    "write.method": "upsert",
    "transforms": "extractKey,convertDate",
    "transforms.extractKey.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
    "transforms.extractKey.field": "id",
    "transforms.convertDate.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
    "transforms.convertDate.field": "created_at,updated_at",
    "transforms.convertDate.target.type": "Timestamp",
    "transforms.convertDate.format": "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'",
    "errors.tolerance": "all",
    "errors.log.enable": "true",
    "errors.deadletterqueue.topic.name": "dlq-elasticsearch-sink",
    "batch.size": "100",
    "max.in.flight.requests": "5",
    "max.buffered.records": "10000",
    "linger.ms": "100",
    "flush.timeout.ms": "30000",
    "read.timeout.ms": "30000",
    "connection.timeout.ms": "10000"
  }
}
```

### 4.2 索引模板配置

```json
// elasticsearch/index-templates/users.json
{
  "index_patterns": ["users*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "analysis": {
        "analyzer": {
          "autocomplete": {
            "tokenizer": "autocomplete_tokenizer",
            "filter": ["lowercase"]
          },
          "autocomplete_search": {
            "tokenizer": "lowercase"
          }
        },
        "tokenizer": {
          "autocomplete_tokenizer": {
            "type": "edge_ngram",
            "min_gram": 2,
            "max_gram": 20,
            "token_chars": ["letter", "digit"]
          }
        }
      }
    },
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "id": { "type": "integer" },
        "username": {
          "type": "text",
          "analyzer": "autocomplete",
          "search_analyzer": "autocomplete_search",
          "fields": {
            "keyword": { "type": "keyword" }
          }
        },
        "email": {
          "type": "keyword"
        },
        "full_name": {
          "type": "text",
          "analyzer": "standard"
        },
        "avatar_url": { "type": "keyword" },
        "status": { "type": "keyword" },
        "created_at": { "type": "date" },
        "updated_at": { "type": "date" },
        "__op": { "type": "keyword" },
        "__ts_ms": { "type": "date" }
      }
    }
  }
}
```

```json
// elasticsearch/index-templates/products.json
{
  "index_patterns": ["products*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 0,
      "analysis": {
        "analyzer": {
          "custom_analyzer": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": ["lowercase", "asciifolding", "synonym_filter"]
          }
        },
        "filter": {
          "synonym_filter": {
            "type": "synonym_graph",
            "synonyms": [
              "wireless, bluetooth, cordless",
              "laptop, notebook, computer"
            ]
          }
        }
      }
    },
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "id": { "type": "integer" },
        "name": {
          "type": "text",
          "analyzer": "custom_analyzer",
          "fields": {
            "keyword": { "type": "keyword" },
            "suggest": {
              "type": "completion"
            }
          }
        },
        "description": {
          "type": "text",
          "analyzer": "custom_analyzer"
        },
        "price": { "type": "scaled_float", "scaling_factor": 100 },
        "category": { "type": "keyword" },
        "tags": { "type": "keyword" },
        "inventory_count": { "type": "integer" },
        "is_active": { "type": "boolean" },
        "metadata": { "type": "object", "enabled": true },
        "created_at": { "type": "date" },
        "updated_at": { "type": "date" },
        "__op": { "type": "keyword" },
        "__ts_ms": { "type": "date" }
      }
    }
  }
}
```

---

## 5. 数据一致性保障

### 5.1 一致性策略

```python
# src/consistency/checker.py
"""
数据一致性检查和修复模块
"""
import asyncio
import logging
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
import asyncpg
from elasticsearch import AsyncElasticsearch
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass
class ConsistencyCheckResult:
    table: str
    total_postgres: int
    total_elasticsearch: int
    missing_in_es: List[Dict]
    extra_in_es: List[str]
    mismatched: List[Dict]
    check_time: datetime


class ConsistencyChecker:
    """PostgreSQL 与 Elasticsearch 数据一致性检查器"""
    
    def __init__(
        self,
        pg_dsn: str,
        es_hosts: List[str],
        check_interval: int = 3600  # 默认每小时检查一次
    ):
        self.pg_dsn = pg_dsn
        self.es_client = AsyncElasticsearch(hosts=es_hosts)
        self.check_interval = check_interval
        self._running = False
    
    async def check_table_consistency(
        self,
        table_name: str,
        es_index: str,
        primary_key: str = "id",
        batch_size: int = 1000
    ) -> ConsistencyCheckResult:
        """
        检查单个表的数据一致性
        
        Args:
            table_name: PostgreSQL 表名
            es_index: Elasticsearch 索引名
            primary_key: 主键字段名
            batch_size: 批量处理大小
        """
        pg_pool = await asyncpg.create_pool(self.pg_dsn)
        
        try:
            # 1. 获取 PostgreSQL 数据计数
            async with pg_pool.acquire() as conn:
                pg_count = await conn.fetchval(
                    f"SELECT COUNT(*) FROM {table_name}"
                )
            
            # 2. 获取 Elasticsearch 数据计数
            es_count = await self.es_client.count(index=es_index)
            es_total = es_count["count"]
            
            missing_in_es = []
            extra_in_es = []
            mismatched = []
            
            # 3. 批量比较数据
            offset = 0
            while offset < pg_count:
                async with pg_pool.acquire() as conn:
                    rows = await conn.fetch(
                        f"SELECT * FROM {table_name} ORDER BY {primary_key} LIMIT $1 OFFSET $2",
                        batch_size, offset
                    )
                
                for row in rows:
                    row_dict = dict(row)
                    pk_value = row_dict[primary_key]
                    
                    # 查询 Elasticsearch
                    try:
                        es_doc = await self.es_client.get(
                            index=es_index, id=str(pk_value)
                        )
                        es_source = es_doc["_source"]
                        
                        # 比较字段
                        mismatch_fields = self._compare_records(
                            row_dict, es_source, table_name
                        )
                        if mismatch_fields:
                            mismatched.append({
                                "id": pk_value,
                                "fields": mismatch_fields
                            })
                    except Exception as e:
                        if "not found" in str(e).lower():
                            missing_in_es.append(row_dict)
                        else:
                            logger.error(f"Error querying ES for {pk_value}: {e}")
                
                offset += batch_size
            
            # 4. 检查 Elasticsearch 中多余的记录
            # 扫描 ES 中所有文档，检查是否存在于 PG
            await self._check_extra_documents(
                es_index, table_name, primary_key, pg_pool, extra_in_es
            )
            
            return ConsistencyCheckResult(
                table=table_name,
                total_postgres=pg_count,
                total_elasticsearch=es_total,
                missing_in_es=missing_in_es,
                extra_in_es=extra_in_es,
                mismatched=mismatched,
                check_time=datetime.utcnow()
            )
            
        finally:
            await pg_pool.close()
    
    def _compare_records(
        self,
        pg_record: Dict,
        es_record: Dict,
        table_name: str
    ) -> List[Dict]:
        """比较两条记录，返回不匹配的字段"""
        mismatches = []
        
        # 定义字段映射（处理字段名差异）
        field_mappings = {
            "created_at": ["created_at", "__ts_ms"],
            "updated_at": ["updated_at"]
        }
        
        for pg_field, pg_value in pg_record.items():
            es_field = pg_field
            es_value = es_record.get(es_field)
            
            # 跳过 CDC 元数据字段
            if pg_field.startswith("__"):
                continue
            
            # 标准化比较
            if not self._values_equal(pg_value, es_value):
                mismatches.append({
                    "field": pg_field,
                    "postgres": str(pg_value),
                    "elasticsearch": str(es_value)
                })
        
        return mismatches
    
    def _values_equal(self, pg_value, es_value) -> bool:
        """比较两个值是否相等，处理类型差异"""
        if pg_value is None and es_value is None:
            return True
        if pg_value is None or es_value is None:
            return False
        
        # 处理 datetime
        if isinstance(pg_value, datetime):
            # 将 ES 的时间戳字符串转换为 datetime
            if isinstance(es_value, str):
                try:
                    es_dt = datetime.fromisoformat(es_value.replace('Z', '+00:00'))
                    return abs((pg_value - es_dt).total_seconds()) < 1
                except:
                    return False
        
        # 处理 JSON/数组
        if isinstance(pg_value, (list, dict)):
            return str(pg_value) == str(es_value)
        
        return str(pg_value) == str(es_value)
    
    async def _check_extra_documents(
        self,
        es_index: str,
        table_name: str,
        primary_key: str,
        pg_pool: asyncpg.Pool,
        extra_in_es: List[str]
    ):
        """检查 Elasticsearch 中多余的文档"""
        query = {
            "query": {"match_all": {}},
            "_source": False,
            "size": 10000
        }
        
        try:
            response = await self.es_client.search(index=es_index, body=query)
            for hit in response["hits"]["hits"]:
                doc_id = hit["_id"]
                
                # 检查是否存在于 PostgreSQL
                async with pg_pool.acquire() as conn:
                    exists = await conn.fetchval(
                        f"SELECT 1 FROM {table_name} WHERE {primary_key} = $1",
                        int(doc_id) if doc_id.isdigit() else doc_id
                    )
                    if not exists:
                        extra_in_es.append(doc_id)
        except Exception as e:
            logger.error(f"Error checking extra documents: {e}")
    
    async def repair_inconsistencies(
        self,
        result: ConsistencyCheckResult,
        es_index: str,
        pg_pool: asyncpg.Pool
    ) -> Dict:
        """
        修复数据不一致
        
        Returns:
            修复操作的结果统计
        """
        repair_stats = {
            "added": 0,
            "deleted": 0,
            "updated": 0,
            "failed": 0
        }
        
        # 1. 修复缺失的记录（从 PG 同步到 ES）
        for record in result.missing_in_es:
            try:
                doc_id = str(record["id"])
                await self.es_client.index(
                    index=es_index,
                    id=doc_id,
                    document=record
                )
                repair_stats["added"] += 1
            except Exception as e:
                logger.error(f"Failed to add record {record.get('id')}: {e}")
                repair_stats["failed"] += 1
        
        # 2. 删除多余的记录
        for doc_id in result.extra_in_es:
            try:
                await self.es_client.delete(index=es_index, id=doc_id)
                repair_stats["deleted"] += 1
            except Exception as e:
                logger.error(f"Failed to delete document {doc_id}: {e}")
                repair_stats["failed"] += 1
        
        # 3. 更新不匹配的字段
        for mismatch in result.mismatched:
            try:
                doc_id = str(mismatch["id"])
                # 从 PG 重新获取完整记录
                async with pg_pool.acquire() as conn:
                    row = await conn.fetchrow(
                        f"SELECT * FROM {result.table} WHERE id = $1",
                        mismatch["id"]
                    )
                    if row:
                        await self.es_client.index(
                            index=es_index,
                            id=doc_id,
                            document=dict(row)
                        )
                        repair_stats["updated"] += 1
            except Exception as e:
                logger.error(f"Failed to update record {mismatch['id']}: {e}")
                repair_stats["failed"] += 1
        
        return repair_stats


class ConsistencyMonitor:
    """一致性监控服务"""
    
    def __init__(self, checker: ConsistencyChecker):
        self.checker = checker
        self.alert_threshold = 0.01  # 1% 差异率触发告警
    
    async def continuous_monitor(
        self,
        tables_config: List[Dict],
        on_inconsistency: Optional[callable] = None
    ):
        """持续监控数据一致性"""
        while self.checker._running:
            for config in tables_config:
                result = await self.checker.check_table_consistency(
                    table_name=config["table"],
                    es_index=config["index"],
                    primary_key=config.get("primary_key", "id")
                )
                
                # 计算差异率
                total_diff = (
                    len(result.missing_in_es) +
                    len(result.extra_in_es) +
                    len(result.mismatched)
                )
                diff_rate = total_diff / max(result.total_postgres, 1)
                
                logger.info(
                    f"Consistency check for {result.table}: "
                    f"PG={result.total_postgres}, ES={result.total_elasticsearch}, "
                    f"Diff={diff_rate:.2%}"
                )
                
                # 触发告警
                if diff_rate > self.alert_threshold and on_inconsistency:
                    await on_inconsistency(result)
            
            await asyncio.sleep(self.checker.check_interval)


# 使用示例
async def main():
    checker = ConsistencyChecker(
        pg_dsn="postgresql://app_user:secure_password@localhost:5432/app_database",
        es_hosts=["http://localhost:9200"],
        check_interval=3600
    )
    
    # 检查 users 表
    result = await checker.check_table_consistency(
        table_name="users",
        es_index="users"
    )
    
    print(f"一致性检查结果:")
    print(f"  PostgreSQL 记录数: {result.total_postgres}")
    print(f"  Elasticsearch 记录数: {result.total_elasticsearch}")
    print(f"  ES 中缺失: {len(result.missing_in_es)}")
    print(f"  ES 中多余: {len(result.extra_in_es)}")
    print(f"  字段不匹配: {len(result.mismatched)}")


if __name__ == "__main__":
    asyncio.run(main())
```

### 5.2 数据校验 API

```python
# src/api/consistency_api.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import asyncio

from ..consistency.checker import ConsistencyChecker, ConsistencyCheckResult

app = FastAPI(title="Data Consistency API")


class ConsistencyCheckRequest(BaseModel):
    table: str
    index: str
    primary_key: str = "id"
    auto_repair: bool = False


class ConsistencyCheckResponse(BaseModel):
    table: str
    postgres_count: int
    elasticsearch_count: int
    missing_count: int
    extra_count: int
    mismatch_count: int
    consistency_rate: float
    repaired: Optional[dict] = None


# 初始化检查器
checker = ConsistencyChecker(
    pg_dsn="postgresql://app_user:secure_password@postgres:5432/app_database",
    es_hosts=["http://elasticsearch:9200"]
)


@app.post("/api/v1/consistency/check", response_model=ConsistencyCheckResponse)
async def check_consistency(request: ConsistencyCheckRequest):
    """执行数据一致性检查"""
    try:
        result = await checker.check_table_consistency(
            table_name=request.table,
            es_index=request.index,
            primary_key=request.primary_key
        )
        
        response = ConsistencyCheckResponse(
            table=result.table,
            postgres_count=result.total_postgres,
            elasticsearch_count=result.total_elasticsearch,
            missing_count=len(result.missing_in_es),
            extra_count=len(result.extra_in_es),
            mismatch_count=len(result.mismatched),
            consistency_rate=1.0 - (
                len(result.missing_in_es) + len(result.extra_in_es) + len(result.mismatched)
            ) / max(result.total_postgres, 1)
        )
        
        # 自动修复
        if request.auto_repair and (
            result.missing_in_es or result.extra_in_es or result.mismatched
        ):
            import asyncpg
            pg_pool = await asyncpg.create_pool(checker.pg_dsn)
            try:
                repair_stats = await checker.repair_inconsistencies(
                    result, request.index, pg_pool
                )
                response.repaired = repair_stats
            finally:
                await pg_pool.close()
        
        return response
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.get("/api/v1/consistency/status")
async def get_consistency_status():
    """获取所有表的总体一致性状态"""
    tables = [
        {"table": "users", "index": "users"},
        {"table": "products", "index": "products"},
        {"table": "orders", "index": "orders"}
    ]
    
    results = []
    for config in tables:
        result = await checker.check_table_consistency(
            table_name=config["table"],
            es_index=config["index"]
        )
        total_diff = (
            len(result.missing_in_es) +
            len(result.extra_in_es) +
            len(result.mismatched)
        )
        results.append({
            "table": config["table"],
            "status": "consistent" if total_diff == 0 else "inconsistent",
            "postgres_count": result.total_postgres,
            "elasticsearch_count": result.total_elasticsearch,
            "difference_count": total_diff
        })
    
    return {
        "overall_status": "healthy" if all(r["status"] == "consistent" for r in results) else "warning",
        "tables": results
    }
```

---

## 6. 错误恢复机制

### 6.1 死信队列处理

```python
# src/recovery/dead_letter_handler.py
"""
死信队列处理器 - 处理同步失败的消息
"""
import json
import logging
from datetime import datetime
from typing import Dict, Callable, Optional
from kafka import KafkaConsumer, KafkaProducer, TopicPartition
from elasticsearch import Elasticsearch
import backoff

logger = logging.getLogger(__name__)


class DeadLetterQueueHandler:
    """死信队列处理器"""
    
    def __init__(
        self,
        bootstrap_servers: str,
        dlq_topic: str,
        es_hosts: List[str],
        max_retries: int = 3
    ):
        self.bootstrap_servers = bootstrap_servers
        self.dlq_topic = dlq_topic
        self.es_client = Elasticsearch(hosts=es_hosts)
        self.max_retries = max_retries
        
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None
        )
        
        self.retry_handlers: Dict[str, Callable] = {}
    
    def register_retry_handler(self, error_type: str, handler: Callable):
        """注册错误类型的重试处理器"""
        self.retry_handlers[error_type] = handler
    
    @backoff.on_exception(
        backoff.expo,
        Exception,
        max_tries=3,
        giveup=lambda e: "permanent" in str(e).lower()
    )
    async def process_dlq_message(self, message: Dict) -> bool:
        """
        处理死信队列中的消息
        
        Returns:
            True if processed successfully, False otherwise
        """
        try:
            original_topic = message.get("original_topic")
            error_info = message.get("error", {})
            error_type = error_info.get("type", "unknown")
            original_value = message.get("value", {})
            
            logger.info(f"Processing DLQ message from {original_topic}, error: {error_type}")
            
            # 根据错误类型选择处理策略
            if error_type in self.retry_handlers:
                return await self.retry_handlers[error_type](message)
            
            # 默认处理：尝试重新索引到 ES
            return await self._retry_index(message)
            
        except Exception as e:
            logger.error(f"Failed to process DLQ message: {e}")
            return False
    
    async def _retry_index(self, message: Dict) -> bool:
        """重试索引到 Elasticsearch"""
        try:
            original_value = message.get("value", {})
            index = message.get("target_index")
            doc_id = message.get("key")
            
            # 清理可能导致错误的数据
            cleaned_doc = self._sanitize_document(original_value)
            
            # 尝试索引
            self.es_client.index(
                index=index,
                id=doc_id,
                document=cleaned_doc
            )
            
            logger.info(f"Successfully re-indexed document {doc_id} to {index}")
            return True
            
        except Exception as e:
            logger.error(f"Retry index failed: {e}")
            # 如果重试次数超过限制，归档到长期存储
            if message.get("retry_count", 0) >= self.max_retries:
                await self._archive_failed_message(message)
            return False
    
    def _sanitize_document(self, doc: Dict) -> Dict:
        """清理文档中的无效数据"""
        cleaned = {}
        for key, value in doc.items():
            # 处理 NaN 和 Infinity
            if isinstance(value, float):
                if value != value:  # NaN check
                    cleaned[key] = None
                elif value == float('inf') or value == float('-inf'):
                    cleaned[key] = None
                else:
                    cleaned[key] = value
            # 递归处理嵌套对象
            elif isinstance(value, dict):
                cleaned[key] = self._sanitize_document(value)
            elif isinstance(value, list):
                cleaned[key] = [
                    self._sanitize_document(item) if isinstance(item, dict) else item
                    for item in value
                ]
            else:
                cleaned[key] = value
        return cleaned
    
    async def _archive_failed_message(self, message: Dict):
        """归档永久失败的消息到单独索引"""
        try:
            archive_doc = {
                "original_message": message,
                "archived_at": datetime.utcnow().isoformat(),
                "archive_reason": "max_retries_exceeded"
            }
            
            self.es_client.index(
                index="failed-sync-archive",
                document=archive_doc
            )
            
            logger.warning(f"Archived failed message after max retries")
            
        except Exception as e:
            logger.error(f"Failed to archive message: {e}")
    
    def republish_to_source(self, message: Dict, target_topic: str):
        """重新发布消息到原始 Topic"""
        try:
            key = message.get("key")
            value = message.get("value")
            
            self.producer.send(
                topic=target_topic,
                key=key.encode() if key else None,
                value=value
            )
            
            logger.info(f"Republished message to {target_topic}")
            
        except Exception as e:
            logger.error(f"Failed to republish message: {e}")
    
    async def run_consumer(self):
        """运行 DLQ 消费者"""
        consumer = KafkaConsumer(
            self.dlq_topic,
            bootstrap_servers=self.bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            group_id="dlq-processor",
            auto_offset_reset="earliest"
        )
        
        logger.info(f"Started DLQ consumer for topic: {self.dlq_topic}")
        
        for message in consumer:
            try:
                success = await self.process_dlq_message(message.value)
                if success:
                    # 处理成功，提交 offset
                    consumer.commit()
                else:
                    # 处理失败，增加重试计数并决定是否重新发布
                    msg_value = message.value
                    msg_value["retry_count"] = msg_value.get("retry_count", 0) + 1
                    
                    if msg_value["retry_count"] < self.max_retries:
                        # 延迟后重新发布到 DLQ
                        await asyncio.sleep(2 ** msg_value["retry_count"])  # 指数退避
                        self.republish_to_source(msg_value, self.dlq_topic)
                    
            except Exception as e:
                logger.error(f"Error processing DLQ message: {e}")


class CircuitBreaker:
    """
    熔断器模式 - 防止级联故障
    """
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 60,
        half_open_max_calls: int = 3
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_calls = half_open_max_calls
        
        self.failures = 0
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
        self.half_open_calls = 0
    
    def can_execute(self) -> bool:
        """检查是否允许执行操作"""
        if self.state == "CLOSED":
            return True
        
        if self.state == "OPEN":
            # 检查是否过了恢复时间
            if self.last_failure_time:
                elapsed = (datetime.utcnow() - self.last_failure_time).seconds
                if elapsed >= self.recovery_timeout:
                    self.state = "HALF_OPEN"
                    self.half_open_calls = 0
                    logger.info("Circuit breaker entering HALF_OPEN state")
                    return True
            return False
        
        if self.state == "HALF_OPEN":
            if self.half_open_calls < self.half_open_max_calls:
                self.half_open_calls += 1
                return True
            return False
        
        return True
    
    def record_success(self):
        """记录成功调用"""
        if self.state == "HALF_OPEN":
            self.state = "CLOSED"
            self.failures = 0
            self.half_open_calls = 0
            logger.info("Circuit breaker CLOSED")
        elif self.state == "CLOSED":
            self.failures = max(0, self.failures - 1)
    
    def record_failure(self):
        """记录失败调用"""
        self.failures += 1
        self.last_failure_time = datetime.utcnow()
        
        if self.state == "HALF_OPEN":
            self.state = "OPEN"
            logger.warning("Circuit breaker OPEN (half-open call failed)")
        elif self.state == "CLOSED" and self.failures >= self.failure_threshold:
            self.state = "OPEN"
            logger.warning(f"Circuit breaker OPEN after {self.failures} failures")


# 带熔断器的 ES 写入包装器
class ResilientESWriter:
    """具有熔断器保护的 Elasticsearch 写入器"""
    
    def __init__(self, es_hosts: List[str]):
        self.es_client = Elasticsearch(hosts=es_hosts)
        self.circuit_breaker = CircuitBreaker()
        self.dlq_handler: Optional[DeadLetterQueueHandler] = None
    
    def set_dlq_handler(self, handler: DeadLetterQueueHandler):
        self.dlq_handler = handler
    
    async def index(self, index: str, doc_id: str, document: Dict) -> bool:
        """带熔断保护的索引操作"""
        if not self.circuit_breaker.can_execute():
            # 熔断器打开，发送到 DLQ
            if self.dlq_handler:
                self.dlq_handler.producer.send(
                    "dlq-elasticsearch-sink",
                    value={
                        "action": "index",
                        "target_index": index,
                        "key": doc_id,
                        "value": document,
                        "error": {"type": "circuit_breaker_open"},
                        "timestamp": datetime.utcnow().isoformat()
                    }
                )
            return False
        
        try:
            self.es_client.index(index=index, id=doc_id, document=document)
            self.circuit_breaker.record_success()
            return True
            
        except Exception as e:
            self.circuit_breaker.record_failure()
            
            # 发送到 DLQ
            if self.dlq_handler:
                self.dlq_handler.producer.send(
                    "dlq-elasticsearch-sink",
                    value={
                        "action": "index",
                        "target_index": index,
                        "key": doc_id,
                        "value": document,
                        "error": {"type": type(e).__name__, "message": str(e)},
                        "timestamp": datetime.utcnow().isoformat()
                    }
                )
            
            return False
```

---

## 7. 监控与告警

### 7.1 Prometheus 配置

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "cdc_alerts.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'kafka-connect'
    static_configs:
      - targets: ['kafka-connect:8083']
    metrics_path: '/metrics'

  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka:9999']

  - job_name: 'elasticsearch'
    static_configs:
      - targets: ['elasticsearch:9114']

  - job_name: 'postgres-exporter'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

### 7.2 告警规则

```yaml
# monitoring/cdc_alerts.yml
groups:
  - name: cdc_alerts
    rules:
      # Kafka Connect 连接器失败
      - alert: KafkaConnectConnectorFailed
        expr: kafka_connect_connector_status{status="FAILED"} == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka Connect connector {{ $labels.connector }} has failed"
          description: "Connector {{ $labels.connector }} is in FAILED state"

      # 复制延迟过高
      - alert: CDCReplicationLag
        expr: debezium_cdc_lag_seconds > 300
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CDC replication lag is high"
          description: "Replication lag is {{ $value }} seconds"

      # 死信队列堆积
      - alert: DeadLetterQueueBacklog
        expr: kafka_consumer_lag{topic=~"dlq-.*"} > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Dead letter queue has backlog"
          description: "DLQ {{ $labels.topic }} has {{ $value }} messages pending"

      # Elasticsearch 索引速率下降
      - alert: ESIndexingRateDrop
        expr: rate(elasticsearch_indices_indexing_index_total[5m]) < 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Elasticsearch indexing rate is low"
          description: "Indexing rate dropped to {{ $value }} docs/sec"

      # 数据不一致告警
      - alert: DataInconsistencyDetected
        expr: cdc_consistency_difference_count > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Data inconsistency detected"
          description: "Table {{ $labels.table }} has {{ $value }} inconsistent records"

      # PostgreSQL 复制槽问题
      - alert: PostgresReplicationSlotInactive
        expr: pg_replication_slot_active == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL replication slot is inactive"
          description: "Replication slot {{ $labels.slot_name }} is not active"
```

### 7.3 自定义指标收集器

```python
# src/monitoring/metrics_collector.py
"""
CDC 系统监控指标收集器
"""
import asyncio
import logging
from datetime import datetime
from typing import Dict, Optional
from prometheus_client import Counter, Gauge, Histogram, Info, start_http_server
import aiohttp

logger = logging.getLogger(__name__)


class CDCMetricsCollector:
    """CDC 系统指标收集器"""
    
    def __init__(self, port: int = 9999):
        self.port = port
        
        # 连接器状态
        self.connector_status = Gauge(
            'cdc_connector_status',
            'Status of CDC connectors (1=RUNNING, 0=FAILED, 2=PAUSED)',
            ['connector_name', 'type']  # type: source or sink
        )
        
        # 复制延迟
        self.replication_lag = Gauge(
            'cdc_replication_lag_seconds',
            'Replication lag in seconds',
            ['source_connector', 'table']
        )
        
        # 处理的消息数
        self.messages_processed = Counter(
            'cdc_messages_processed_total',
            'Total number of messages processed',
            ['connector', 'table', 'operation']  # operation: c, u, d
        )
        
        # 消息处理延迟
        self.message_processing_duration = Histogram(
            'cdc_message_processing_duration_seconds',
            'Time taken to process messages',
            ['connector', 'table'],
            buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
        )
        
        # 错误计数
        self.errors_total = Counter(
            'cdc_errors_total',
            'Total number of errors',
            ['connector', 'error_type']
        )
        
        # 死信队列大小
        self.dlq_size = Gauge(
            'cdc_dlq_messages',
            'Number of messages in dead letter queue',
            ['topic']
        )
        
        # 数据一致性指标
        self.consistency_difference = Gauge(
            'cdc_consistency_difference_count',
            'Number of records with differences between source and target',
            ['table', 'index']
        )
        
        self.consistency_check_duration = Histogram(
            'cdc_consistency_check_duration_seconds',
            'Time taken to perform consistency check',
            ['table']
        )
        
        # 系统信息
        self.system_info = Info(
            'cdc_system',
            'CDC system information'
        )
        
        # 最后检查时间
        self.last_check_timestamp = Gauge(
            'cdc_last_check_timestamp',
            'Timestamp of last check',
            ['check_type']
        )
    
    def start_server(self):
        """启动 Prometheus HTTP 服务器"""
        start_http_server(self.port)
        logger.info(f"Metrics server started on port {self.port}")
    
    async def collect_kafka_connect_metrics(
        self,
        connect_url: str = "http://localhost:8083"
    ):
        """收集 Kafka Connect 指标"""
        async with aiohttp.ClientSession() as session:
            try:
                # 获取所有连接器
                async with session.get(f"{connect_url}/connectors") as resp:
                    if resp.status == 200:
                        connectors = await resp.json()
                        
                        for connector in connectors:
                            # 获取连接器状态
                            async with session.get(
                                f"{connect_url}/connectors/{connector}/status"
                            ) as status_resp:
                                if status_resp.status == 200:
                                    status_data = await status_resp.json()
                                    state = status_data.get("connector", {}).get("state", "UNKNOWN")
                                    
                                    # 确定类型
                                    config_resp = await session.get(
                                        f"{connect_url}/connectors/{connector}/config"
                                    )
                                    config = await config_resp.json()
                                    connector_class = config.get("connector.class", "")
                                    
                                    if "postgres" in connector_class.lower():
                                        conn_type = "source"
                                    elif "elasticsearch" in connector_class.lower():
                                        conn_type = "sink"
                                    else:
                                        conn_type = "unknown"
                                    
                                    # 设置指标
                                    status_value = 1 if state == "RUNNING" else (0 if state == "FAILED" else 2)
                                    self.connector_status.labels(
                                        connector_name=connector,
                                        type=conn_type
                                    ).set(status_value)
                                    
                                    # 记录任务级指标
                                    for task in status_data.get("tasks", []):
                                        if task.get("state") == "FAILED":
                                            self.errors_total.labels(
                                                connector=connector,
                                                error_type="task_failure"
                                            ).inc()
                    
            except Exception as e:
                logger.error(f"Error collecting Kafka Connect metrics: {e}")
                self.errors_total.labels(
                    connector="metrics_collector",
                    error_type="collection_error"
                ).inc()
    
    async def collect_replication_lag(
        self,
        pg_dsn: str,
        slot_name: str = "debezium_slot"
    ):
        """收集复制延迟指标"""
        import asyncpg
        
        try:
            conn = await asyncpg.connect(pg_dsn)
            
            # 查询复制槽的延迟
            row = await conn.fetchrow("""
                SELECT 
                    slot_name,
                    confirmed_flush_lsn,
                    pg_current_wal_lsn(),
                    pg_current_wal_lsn() - confirmed_flush_lsn as lag_bytes
                FROM pg_replication_slots 
                WHERE slot_name = $1
            """, slot_name)
            
            if row:
                lag_bytes = row['lag_bytes'] or 0
                # 估算延迟（假设平均 WAL 记录大小为 100 bytes）
                estimated_lag_seconds = lag_bytes / 100 * 0.01
                
                self.replication_lag.labels(
                    source_connector="postgres-source-connector",
                    table="all"
                ).set(estimated_lag_seconds)
            
            await conn.close()
            
        except Exception as e:
            logger.error(f"Error collecting replication lag: {e}")
    
    async def collect_dlq_metrics(
        self,
        bootstrap_servers: str = "localhost:9092"
    ):
        """收集死信队列指标"""
        from kafka import KafkaConsumer
        
        try:
            consumer = KafkaConsumer(
                bootstrap_servers=bootstrap_servers
            )
            
            # 获取 DLQ 主题的偏移量信息
            dlq_topics = [t for t in consumer.topics() if t.startswith("dlq-")]
            
            for topic in dlq_topics:
                partitions = consumer.partitions_for_topic(topic)
                if partitions:
                    total_lag = 0
                    for partition in partitions:
                        tp = TopicPartition(topic, partition)
                        consumer.assign([tp])
                        consumer.seek_to_end(tp)
                        end_offset = consumer.position(tp)
                        consumer.seek_to_beginning(tp)
                        beginning_offset = consumer.position(tp)
                        total_lag += end_offset - beginning_offset
                    
                    self.dlq_size.labels(topic=topic).set(total_lag)
            
            consumer.close()
            
        except Exception as e:
            logger.error(f"Error collecting DLQ metrics: {e}")
    
    async def update_consistency_metrics(
        self,
        checker,
        tables_config: list
    ):
        """更新数据一致性指标"""
        for config in tables_config:
            start_time = datetime.utcnow()
            
            try:
                result = await checker.check_table_consistency(
                    table_name=config["table"],
                    es_index=config["index"]
                )
                
                total_diff = (
                    len(result.missing_in_es) +
                    len(result.extra_in_es) +
                    len(result.mismatched)
                )
                
                self.consistency_difference.labels(
                    table=config["table"],
                    index=config["index"]
                ).set(total_diff)
                
                # 记录检查耗时
                duration = (datetime.utcnow() - start_time).total_seconds()
                self.consistency_check_duration.labels(
                    table=config["table"]
                ).observe(duration)
                
            except Exception as e:
                logger.error(f"Error updating consistency metrics for {config['table']}: {e}")
        
        self.last_check_timestamp.labels(check_type="consistency").set_to_current_time()
    
    async def run_metrics_collection(
        self,
        pg_dsn: str,
        connect_url: str = "http://localhost:8083",
        kafka_servers: str = "localhost:9092",
        consistency_checker = None,
        tables_config: list = None,
        interval: int = 30
    ):
        """持续运行指标收集"""
        self.start_server()
        
        while True:
            try:
                # 收集 Kafka Connect 指标
                await self.collect_kafka_connect_metrics(connect_url)
                
                # 收集复制延迟
                await self.collect_replication_lag(pg_dsn)
                
                # 收集 DLQ 指标
                await self.collect_dlq_metrics(kafka_servers)
                
                # 更新一致性指标（如果提供了检查器）
                if consistency_checker and tables_config:
                    await self.update_consistency_metrics(
                        consistency_checker, tables_config
                    )
                
                self.last_check_timestamp.labels(
                    check_type="metrics_collection"
                ).set_to_current_time()
                
            except Exception as e:
                logger.error(f"Error in metrics collection loop: {e}")
            
            await asyncio.sleep(interval)


# 使用示例
async def main():
    collector = CDCMetricsCollector(port=9999)
    
    await collector.run_metrics_collection(
        pg_dsn="postgresql://app_user:secure_password@localhost:5432/app_database",
        connect_url="http://localhost:8083",
        kafka_servers="localhost:9092",
        interval=30
    )


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 8. 部署与运维脚本

### 8.1 一键启动脚本

```bash
#!/bin/bash
# scripts/start-cdc.sh

set -e

echo "🚀 Starting CDC Infrastructure..."

# 创建必要的目录
mkdir -p postgres elasticsearch monitoring/grafana/dashboards monitoring/grafana/datasources

# 启动基础设施
echo "📦 Starting Docker containers..."
docker-compose up -d

# 等待服务就绪
echo "⏳ Waiting for services to be ready..."
sleep 10

# 等待 PostgreSQL
until docker exec postgres-cdc pg_isready -U app_user > /dev/null 2>&1; do
    echo "Waiting for PostgreSQL..."
    sleep 2
done

# 等待 Kafka Connect
echo "Waiting for Kafka Connect..."
until curl -s http://localhost:8083/ > /dev/null 2>&1; do
    echo "Waiting for Kafka Connect..."
    sleep 5
done

# 等待 Elasticsearch
echo "Waiting for Elasticsearch..."
until curl -s http://localhost:9200/_cluster/health > /dev/null 2>&1; do
    echo "Waiting for Elasticsearch..."
    sleep 5
done

echo "✅ All services are ready!"

# 创建 Elasticsearch 索引模板
echo "🔧 Setting up Elasticsearch index templates..."
curl -X PUT "http://localhost:9200/_index_template/users_template" \
  -H "Content-Type: application/json" \
  -d @elasticsearch/index-templates/users.json

curl -X PUT "http://localhost:9200/_index_template/products_template" \
  -H "Content-Type: application/json" \
  -d @elasticsearch/index-templates/products.json

# 创建 Debezium 连接器
echo "🔗 Creating Debezium connector..."
./scripts/setup-debezium.sh

# 创建 Elasticsearch Sink 连接器
echo "🔗 Creating Elasticsearch Sink connector..."
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @config/elasticsearch-sink-connector.json

echo "🎉 CDC setup complete!"
echo ""
echo "📊 Access points:"
echo "  - PostgreSQL: localhost:5432"
echo "  - Kafka: localhost:9092"
echo "  - Kafka Connect: http://localhost:8083"
echo "  - Elasticsearch: http://localhost:9200"
echo "  - Kibana: http://localhost:5601"
echo "  - Grafana: http://localhost:3000"
```

### 8.2 健康检查脚本

```bash
#!/bin/bash
# scripts/health-check.sh

set -e

ERRORS=0

echo "🏥 CDC System Health Check"
echo "============================"

# 检查 PostgreSQL
if docker exec postgres-cdc pg_isready -U app_user > /dev/null 2>&1; then
    echo "✅ PostgreSQL: Healthy"
else
    echo "❌ PostgreSQL: Unhealthy"
    ERRORS=$((ERRORS + 1))
fi

# 检查 Kafka
if docker exec kafka kafka-broker-api-versions.sh --bootstrap-server localhost:9092 > /dev/null 2>&1; then
    echo "✅ Kafka: Healthy"
else
    echo "❌ Kafka: Unhealthy"
    ERRORS=$((ERRORS + 1))
fi

# 检查 Kafka Connect
if curl -s http://localhost:8083/ | grep -q "version"; then
    echo "✅ Kafka Connect: Healthy"
else
    echo "❌ Kafka Connect: Unhealthy"
    ERRORS=$((ERRORS + 1))
fi

# 检查连接器状态
echo ""
echo "📋 Connector Status:"
curl -s http://localhost:8083/connectors | jq -r '.[]' | while read connector; do
    status=$(curl -s "http://localhost:8083/connectors/$connector/status" | jq -r '.connector.state')
    if [ "$status" = "RUNNING" ]; then
        echo "  ✅ $connector: $status"
    else
        echo "  ❌ $connector: $status"
        ERRORS=$((ERRORS + 1))
    fi
done

# 检查 Elasticsearch
if curl -s http://localhost:9200/_cluster/health | grep -q '"status":"green"\|"status":"yellow"'; then
    echo "✅ Elasticsearch: Healthy"
else
    echo "❌ Elasticsearch: Unhealthy"
    ERRORS=$((ERRORS + 1))
fi

# 检查索引
echo ""
echo "📊 Index Status:"
curl -s "http://localhost:9200/_cat/indices/users,products,orders?v&h=index,docs.count,health" 2>/dev/null || echo "  No indices found"

# 检查复制延迟
echo ""
echo "⏱️  Replication Status:"
docker exec postgres-cdc psql -U app_user -d app_database -c "
SELECT 
    slot_name,
    active,
    pg_size_pretty(pg_current_wal_lsn() - confirmed_flush_lsn) as lag
FROM pg_replication_slots;
" 2>/dev/null || echo "  Unable to check replication status"

# 总结
echo ""
echo "============================"
if [ $ERRORS -eq 0 ]; then
    echo "✅ All checks passed!"
    exit 0
else
    echo "❌ $ERRORS check(s) failed!"
    exit 1
fi
```

### 8.3 数据重同步脚本

```bash
#!/bin/bash
# scripts/resync-table.sh

TABLE_NAME=$1
INDEX_NAME=${2:-$TABLE_NAME}

if [ -z "$TABLE_NAME" ]; then
    echo "Usage: $0 <table_name> [index_name]"
    echo "Example: $0 users users"
    exit 1
fi

echo "🔄 Re-syncing table: $TABLE_NAME to index: $INDEX_NAME"

# 1. 暂停连接器
echo "Pausing connectors..."
curl -X PUT "http://localhost:8083/connectors/postgres-source-connector/pause"
curl -X PUT "http://localhost:8083/connectors/elasticsearch-sink-connector/pause"

# 2. 删除并重建 Elasticsearch 索引
echo "Recreating Elasticsearch index..."
curl -X DELETE "http://localhost:9200/${INDEX_NAME}" 2>/dev/null || true
sleep 2

# 3. 重置连接器偏移量
echo "Resetting connector offsets..."
curl -X DELETE "http://localhost:8083/connectors/postgres-source-connector"
sleep 2

# 4. 重新创建连接器（全量快照模式）
echo "Re-creating source connector with snapshot..."
cat > /tmp/reset-connector.json <<EOF
{
  "name": "postgres-source-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "replication_user",
    "database.password": "replication_pass",
    "database.dbname": "app_database",
    "database.server.name": "postgres-server",
    "table.include.list": "public.${TABLE_NAME}",
    "plugin.name": "pgoutput",
    "publication.name": "dbz_publication",
    "slot.name": "debezium_slot",
    "snapshot.mode": "initial",
    "transforms": "unwrap,router",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "transforms.router.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.router.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.router.replacement": "$3",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false"
  }
}
EOF

curl -X POST "http://localhost:8083/connectors" \
  -H "Content-Type: application/json" \
  -d @/tmp/reset-connector.json

# 5. 重新创建 Sink 连接器
echo "Re-creating sink connector..."
curl -X DELETE "http://localhost:8083/connectors/elasticsearch-sink-connector" 2>/dev/null || true
sleep 2

curl -X POST "http://localhost:8083/connectors" \
  -H "Content-Type: application/json" \
  -d @config/elasticsearch-sink-connector.json

echo ""
echo "✅ Re-sync initiated. Check connector status with:"
echo "  curl http://localhost:8083/connectors/postgres-source-connector/status"
```

---

## 9. 性能优化建议

### 9.1 PostgreSQL 优化

```sql
-- 优化 WAL 设置
ALTER SYSTEM SET wal_buffers = '64MB';
ALTER SYSTEM SET max_wal_size = '2GB';
ALTER SYSTEM SET min_wal_size = '512MB';

-- 优化复制槽
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET max_wal_senders = 10;

-- 应用配置
SELECT pg_reload_conf();
```

### 9.2 Kafka 优化配置

```properties
# Kafka 服务器优化
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# 日志段配置
log.segment.bytes=1073741824
log.retention.hours=168
log.retention.check.interval.ms=300000

# 副本配置
default.replication.factor=1
num.partitions=6
min.insync.replicas=1
```

### 9.3 Elasticsearch 优化

```yaml
# elasticsearch/config/elasticsearch.yml
# 批量处理优化
thread_pool.write.queue_size: 1000
thread_pool.search.queue_size: 1000

# 索引优化
index.refresh_interval: 5s
index.translog.durability: async
index.translog.sync_interval: 5s
```

---

## 10. 项目结构

```
cdc-postgres-elasticsearch/
├── docker-compose.yml              # 基础设施编排
├── .env                            # 环境变量配置
├── README.md                       # 项目说明
│
├── config/                         # 连接器配置
│   ├── postgres-source-connector.json
│   └── elasticsearch-sink-connector.json
│
├── scripts/                        # 运维脚本
│   ├── start-cdc.sh
│   ├── stop-cdc.sh
│   ├── health-check.sh
│   ├── setup-debezium.sh
│   └── resync-table.sh
│
├── postgres/                       # PostgreSQL 初始化
│   └── init.sql
│
├── elasticsearch/                  # ES 索引配置
│   └── index-templates/
│       ├── users.json
│       └── products.json
│
├── src/                            # 源代码
│   ├── consistency/               # 一致性检查
│   │   └── checker.py
│   ├── recovery/                  # 错误恢复
│   │   └── dead_letter_handler.py
│   ├── monitoring/                # 监控指标
│   │   └── metrics_collector.py
│   └── api/                       # API 服务
│       └── consistency_api.py
│
├── monitoring/                     # 监控配置
│   ├── prometheus.yml
│   ├── cdc_alerts.yml
│   └── grafana/
│       ├── dashboards/
│       └── datasources/
│
└── tests/                          # 测试
    ├── test_consistency.py
    └── test_recovery.py
```

---

## 11. 总结

### 方案特点

1. **实时性**: 基于 WAL 的 CDC 捕获，延迟通常在毫秒级
2. **可靠性**: 使用 Kafka 作为中间缓冲，保证数据不丢失
3. **可恢复**: 支持从检查点恢复，自动处理失败消息
4. **可观测**: 完整的监控和告警体系
5. **可维护**: 一键部署，自动化运维脚本

### 生产环境建议

1. **高可用**: 部署 Kafka 集群、ES 集群、PostgreSQL 主从
2. **安全**: 启用 SSL/TLS、身份认证、网络隔离
3. **备份**: 定期备份 Kafka offset、ES 快照
4. **限流**: 配置 Kafka Connect 的吞吐限制
5. **扩容**: 根据数据量水平扩展消费者

### 参考文档

- [Debezium Documentation](https://debezium.io/documentation/)
- [Kafka Connect Elasticsearch Sink](https://docs.confluent.io/kafka-connectors/elasticsearch/current/overview.html)
- [PostgreSQL Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
