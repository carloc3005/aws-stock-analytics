# AWS Stock Market Analytics Pipeline - Architecture

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AWS STOCK MARKET ANALYTICS PIPELINE                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐
│   Data Source    │
│                  │
│  yfinance API    │◄─── Fetches real-time stock data (AAPL, GOOGL, etc.)
│  (Yahoo Finance) │
└────────┬─────────┘
         │
         │ Stock Data (JSON)
         │ Every 30 seconds
         ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                          LOCAL PRODUCER SCRIPT                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  carloc_streamstock_data.py                                         │  │
│  │  • Fetches: Open, High, Low, Close, Volume, Previous Close          │  │
│  │  • Computes: Price Change, Change Percent                           │  │
│  │  • Formats: JSON with timestamp                                     │  │
│  │  • Sends to: Amazon Kinesis Data Streams                            │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────┬──────────────────────────────────────────┘
                                  │
                                  │ PUT Record
                                  │ (boto3.kinesis.put_record)
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AWS KINESIS DATA STREAMS                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Stream: carloc-stock-market-stream                                  │  │
│  │  • Shards: 1 (scales to thousands)                                   │  │
│  │  • Retention: 24 hours                                               │  │
│  │  • Throughput: 1MB/sec or 1000 records/sec per shard                │  │
│  │  • Partition Key: Stock Symbol (AAPL)                                │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 │ Event Trigger
                                 │ (Batch Size: 10 records)
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AWS LAMBDA FUNCTION                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Function: stock-data-processor                                      │  │
│  │  Runtime: Python 3.9                                                 │  │
│  │  Memory: 128 MB | Timeout: 60 seconds                                │  │
│  │                                                                       │  │
│  │  PROCESSING LOGIC:                                                   │  │
│  │  1. Decode base64 Kinesis data                                       │  │
│  │  2. Parse JSON payload                                               │  │
│  │  3. Compute Metrics:                                                 │  │
│  │     • Price Change & Change Percent                                  │  │
│  │     • Moving Average (Open+High+Low+Close)/4                         │  │
│  │     • Anomaly Detection (>5% change)                                 │  │
│  │  4. Convert float → Decimal (DynamoDB compatibility)                 │  │
│  │  5. Parallel Write to:                                               │  │
│  │     ├─ DynamoDB (processed data)                                     │  │
│  │     ├─ S3 (raw JSON data)                                            │  │
│  │     └─ SNS (if anomaly detected)                                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────┬────────────────────────┬────────────────────────┬─────────────────────┘
      │                        │                        │
      │                        │                        │ (Optional Alert)
      ▼                        ▼                        ▼
┌──────────────┐      ┌──────────────────┐    ┌─────────────────┐
│   DYNAMODB   │      │   AMAZON S3      │    │   AMAZON SNS    │
│              │      │                  │    │                 │
│  Table:      │      │  Bucket:         │    │  Topic:         │
│  stock-      │      │  stock-market-   │    │  stock-alerts   │
│  market-data │      │  data-bucket-    │    │                 │
│              │      │  33454           │    │  Subscribers:   │
│  Keys:       │      │                  │    │  • Email        │
│  • symbol    │      │  Structure:      │    │  • SMS          │
│  • timestamp │      │  raw-data/       │    │  • Lambda       │
│              │      │    ├─AAPL/       │    └─────────────────┘
│  Attributes: │      │    │  └─2026...   │
│  • open      │      │    ├─GOOGL/      │
│  • high      │      │    └─MSFT/       │
│  • low       │      │                  │
│  • price     │      │  Format:         │
│  • volume    │      │  • JSON          │
│  • change    │      │  • Partitioned   │
│  • change_%  │      │    by symbol     │
│  • moving_   │      │  • Timestamped   │
│    avg       │      │    files         │
│  • anomaly   │      │                  │
│              │      └────────┬─────────┘
│  Billing:    │               │
│  PAY_PER_    │               │
│  REQUEST     │               │
└──────────────┘               │
                               │ Data Source
                               ▼
                      ┌──────────────────┐
                      │   AWS GLUE       │
                      │                  │
                      │  Crawler:        │
                      │  • Discovers     │
                      │    schema        │
                      │  • Creates       │
                      │    table def     │
                      │                  │
                      │  Database:       │
                      │  stock_market_db │
                      │                  │
                      │  Table:          │
                      │  raw_data        │
                      └────────┬─────────┘
                               │
                               │ Metadata Catalog
                               ▼
                      ┌──────────────────┐
                      │  AMAZON ATHENA   │
                      │                  │
                      │  Query Engine:   │
                      │  • Serverless    │
                      │  • SQL           │
                      │  • Presto-based  │
                      │                  │
                      │  Example Query:  │
                      │  SELECT symbol,  │
                      │    AVG(price)    │
                      │  FROM raw_data   │
                      │  GROUP BY symbol │
                      │                  │
                      │  Results: S3     │
                      └──────────────────┘

                    ┌────────────────────┐
                    │  AWS CLOUDWATCH    │
                    │                    │
                    │  Logs:             │
                    │  • Lambda exec     │
                    │  • Kinesis metrics │
                    │  • Error tracking  │
                    │                    │
                    │  Metrics:          │
                    │  • Invocations     │
                    │  • Duration        │
                    │  • Errors          │
                    │  • Throttles       │
                    └────────────────────┘

```

## Data Flow Sequence

### Phase 1: Data Ingestion (Every 30 seconds)
```
1. Python script calls yfinance API
2. Receives stock data (OHLC, Volume)
3. Formats as JSON with timestamp
4. Sends to Kinesis via boto3.put_record()
```

### Phase 2: Stream Processing (Real-time, < 1 second)
```
5. Kinesis buffers records in shards
6. Lambda triggered when batch size reached (10 records) or 60 seconds
7. Lambda receives batch of Kinesis events
8. For each record:
   a. Decode base64 → JSON
   b. Compute metrics (change %, moving avg, anomaly)
   c. Convert float to Decimal
```

### Phase 3: Parallel Storage (Simultaneous writes)
```
9a. Write processed data to DynamoDB
    → Fast queries, low latency
    
9b. Write raw JSON to S3
    → Long-term storage, historical analysis
    
9c. Publish to SNS (if anomaly)
    → Real-time alerts to subscribers
```

### Phase 4: Analytics (On-demand)
```
10. Glue Crawler scans S3 bucket
11. Discovers schema from JSON files
12. Creates/updates table in Glue Data Catalog
13. Athena queries data using SQL
14. Results returned from S3 scan
```

## Component Details

### Kinesis Data Streams
- **Purpose**: Real-time data ingestion and buffering
- **Capacity**: 1 MB/sec or 1000 records/sec per shard
- **Retention**: 24 hours (configurable up to 365 days)
- **Ordering**: Guaranteed within partition key (symbol)

### Lambda Function
- **Trigger**: Kinesis stream events
- **Batch Size**: 10 records (configurable 1-10,000)
- **Concurrency**: Auto-scales to handle throughput
- **Execution Time**: ~200ms per batch
- **Error Handling**: Failed records returned to Kinesis for retry

### DynamoDB
- **Purpose**: Low-latency processed data storage
- **Capacity Mode**: On-demand (auto-scaling)
- **Keys**: Partition (symbol) + Sort (timestamp)
- **Read Latency**: Single-digit milliseconds
- **Use Case**: Real-time dashboards, recent data queries

### S3
- **Purpose**: Durable raw data storage (data lake)
- **Structure**: Partitioned by stock symbol
- **Format**: JSON files with timestamp names
- **Storage Class**: Standard (can move to Glacier for archival)
- **Use Case**: Historical analysis, compliance, audit

### Glue & Athena
- **Purpose**: Serverless SQL analytics on S3 data
- **Query Speed**: ~2 seconds for 1 month of data
- **Cost**: $5 per TB scanned (compressed data recommended)
- **Use Case**: Trend analysis, reporting, data exploration

## Security Architecture

```
IAM Roles & Policies:

┌─────────────────────────────────────────┐
│  Lambda Execution Role                  │
│  ├─ AWSLambdaKinesisExecutionRole      │
│  │  (Read from Kinesis + CloudWatch)   │
│  ├─ DynamoDB PutItem permission         │
│  └─ S3 PutObject permission             │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Glue Crawler Role                      │
│  ├─ S3 Read permission                  │
│  └─ Glue Data Catalog write permission  │
└─────────────────────────────────────────┘

Encryption:
• DynamoDB: Encrypted at rest (AWS managed keys)
• S3: Server-side encryption (SSE-S3)
• Kinesis: Encrypted in transit (TLS)
• Lambda: Environment variables encrypted
```

## Scalability Considerations

| Component | Current | Max Scale | How to Scale |
|-----------|---------|-----------|--------------|
| Kinesis | 1 shard | 1000s | Add shards, use enhanced fan-out |
| Lambda | Auto | 1000 concurrent | Set concurrency limits |
| DynamoDB | On-demand | Unlimited | Auto-scales with traffic |
| S3 | 50 MB | Exabytes | Automatic scaling |
| Athena | 20 queries | 20/account | Request limit increase |

## Cost Optimization Strategies

1. **Use Free Tier**: Lambda, DynamoDB, SNS, Athena (first 5GB)
2. **On-Demand Billing**: Pay only for actual usage
3. **Minimal Kinesis Shards**: Start with 1, scale as needed
4. **S3 Lifecycle Policies**: Move old data to Glacier
5. **Athena Query Optimization**: Partition data, use columnar formats
6. **Lambda Memory Tuning**: Use minimum required (128 MB)

## Monitoring & Observability

**CloudWatch Metrics to Monitor:**
- Kinesis: `IncomingRecords`, `GetRecords.Latency`
- Lambda: `Invocations`, `Duration`, `Errors`, `Throttles`
- DynamoDB: `ConsumedWriteCapacityUnits`, `SystemErrors`
- S3: `NumberOfObjects`, `BucketSizeBytes`

**CloudWatch Alarms:**
- Lambda errors > 5% → SNS alert
- Kinesis iterator age > 60 seconds → Processing lag
- DynamoDB throttling → Capacity issue

## Disaster Recovery

- **RTO (Recovery Time Objective)**: < 5 minutes
- **RPO (Recovery Point Objective)**: < 1 minute
- **Backup Strategy**: S3 versioning enabled, DynamoDB point-in-time recovery
- **Multi-Region**: Can replicate Kinesis stream and DynamoDB table

---

**Last Updated**: January 2026  
**Author**: [Your Name]
