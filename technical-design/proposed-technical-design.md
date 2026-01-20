# Proposed Technical Design - On Demand Deletion Platform

## System Architecture

![System Architecture Diagram](../diagrams/On%20Demand%20Data%20Deletion%20Platform.png)

## Core Components

### Deletion Processor

**Architecture**: SQS + Lambda two-stage processing system

**Components**:
- **CustomerDeletionRequests SQS Queue**: Receives one message per customer deletion request, agnostic of backend data stores
- **Customer-Level Lambda Worker**: Processes CustomerDeletionRequests queue messages, fans out to table-level deletion requests in DataStoreDeletionRequests queue
- **DataStoreDeletionRequests SQS Queue**: Stores backend data store deletion requests specific to AWS account ID + technology + deletion strategy + customer ID
- **Table-Level Lambda Workers**: Process individual data store deletion requests

**Technical Decisions**:
- **SQS + Lambda architecture**: Chosen for asynchronous processing, fault tolerance, and cost efficiency at expected scale
- **Two-stage processing**: Customer requests first processed to generate multiple backend deletion tasks, then each backend task processed independently
- **SQS over DynamoDB**: Selected for ~40% lower cost and faster implementation
- **Lambda scaling strategy**: Start with Lambda, migrate to ECS Fargate if compute time exceeds ~7 hours/day
- **Authentication & Authorization**: IAM roles and policies for minimal resource access

### Deletion Target API

**Architecture**: REST API Gateway + Lambda + DynamoDB

**Data Model**:
- **DeletionTargets DynamoDB Table**
  - Partition key: `"ACCT#<awsAccountId>#TECH#<technology>"` (e.g., `"ACCT#123456789012#TECH#dynamodb"`)
  - Sort key: deletion target UUID

**Use Cases & Expected Volume**:
- Create deletion target: <1000 transactions/month
- Update deletion target: <1000 transactions/month
- Delete deletion target: <10 transactions/month
- List targets by AWS account + technology: <1000 transactions/month
- Query all targets grouped by account + technology: <1000 transactions/month

**Technical Decisions**:
- **Database**: AWS DynamoDB for cost efficiency (<$1/month), high availability, low maintenance
- **Compute**: AWS Lambda for simplicity and low cost (within free tier)
- **API Interface**: AWS REST API Gateway for native authentication, authorization, metrics, rate limiting, request validation
- **Authentication & Authorization**: IAM with role-based permissions (read-only for deletion workers, read/write for management services)

## API Specifications

### Create Deletion Target API

**Endpoint**: `POST /v1/deletion-platform/deletion-targets`

**Request Schema**: TODO

**Response Schema**:
```json
{
  "deletionTargetId": "string",
  "isEnabled": "boolean"
}
```

### Update Deletion Target API

**Endpoint**: `POST /v1/deletion-platform/deletion-targets/{deletionTargetId}`

**Request Schema**: TODO

**Response Schema**: None

### List Deletion Targets API

**Endpoint**: `GET /v1/deletion-platform/deletion-targets`

**Request Schema**: None

**Response Schema**: TODO

### Delete Deletion Target API

**Endpoint**: `DELETE /v1/deletion-platform/deletion-targets/{deletionTargetId}`

**Request Schema**: None

**Response Schema**: None

## Data Models

### DynamoDB Deletion Target

```kotlin
/**
 * Common data model for on-demand deletion strategies for DynamoDB tables.
 */
data class DynamoDbDeletionTarget(
  val strategy: DynamoDbDeletionStrategyType,
  val awsRegion: String,
  val tableName: String,
  val partitionKeyName: String,
  val sortKeyName: String? = null,
  val deletionKeySchema: DynamoDbDeletionKeySchema,
  val gsiName: String?
)

enum class DynamoDbDeletionStrategy {
  TABLE_KEY,
  GSI_QUERY,
  SCAN
}

/**
 * Data model representing a DynamoDB deletion key schema,
 * which can be used for table key deletions, GSI query-based
 * deletions, or scan-based deletions.
 */
data class DynamoDbDeletionKeySchema(
  val primaryKeyName: String,
  val secondaryKeyName: String? = null
)

/**
 * Common data model for on-demand deletion strategies for S3 buckets.
 */
data class S3DeletionTarget(
  val strategy: S3DeletionStrategyType,
  val awsRegion: String,
  val bucketName: String,
  val objectKeyPrefix: String?,
  val deletionKeyPattern: Pattern?,
  val deletionRowAttributeName: String?,
  val objectFileFormat: String?
)

/**
 * Supported strategies for on-demand data deletion from S3 buckets.
 */
enum class S3DeletionStrategyType {
  OBJECT_KEY,
  ROW_LEVEL
}
```

## Implementation Details

### DynamoDB deletion strategies

#### 1. Table Key Deletion (`TABLE_KEY`)
Delete items by primary key (partition key and optional sort key).

```kotlin
val deletionTarget = DynamoDbDeletionTarget(
    strategy = DynamoDbDeletionStrategyType.TABLE_KEY,
    awsRegion = "us-east-1",
    tableName = "customers",
    partitionKeyName = "serviceId",
    sortKeyName = "customerId", // Optional
    deletionKeySchema = DynamoDbDeletionKeySchema(
        primaryKeyName = "serviceId",
        secondaryKeyName = "customerId" // Optional
    )
)

val deletionKey = DynamoDbDeletionKeyValue(
    primaryKeyValue = "service-123",
    secondaryKeyValue = "customer-123"
)

DynamoDbDeletionConnector(dynamoDbClient).deleteData(deletionTarget, deletionKey)
```

#### 2. GSI Query Deletion (`GSI_QUERY`)
Query a Global Secondary Index and delete all matching items.

```kotlin
val deletionTarget = DynamoDbDeletionTarget(
    strategy = DynamoDbDeletionStrategyType.GSI_QUERY,
    awsRegion = "us-east-1",
    tableName = "purchases",
    partitionKeyName = "purchaseId", // Table partition key
    sortKeyName = "customerId", // Table sort key (optional)
    gsiName = "customer-index",
    deletionKeySchema = DynamoDbDeletionKeySchema(
        primaryKeyName = "customerId", // GSI partition key
        secondaryKeyName = "purchaseId" // GSI sort key (optional)
    )
)

val deletionKey = DynamoDbDeletionKeyValue(
    primaryKeyValue = "customer-123",
    secondaryKeyValue = "0510511e-49d2-4753-8aca-a3ddfc99513b" // Optional
)

DynamoDbDeletionConnector(dynamoDbClient).deleteData(deletionTarget, deletionKey)
```

#### 3. Scan Deletion (`SCAN`)
Scan the entire table and delete items matching specified attributes.

```kotlin
val deletionTarget = DynamoDbDeletionTarget(
    strategy = DynamoDbDeletionStrategyType.SCAN,
    awsRegion = "us-east-1",
    tableName = "purchases",
    partitionKeyName = "purchaseId", // Table partition key
    sortKeyName = "timestamp", // Table sort key (optional)
    deletionKeySchema = DynamoDbDeletionKeySchema(
        primaryKeyName = "tenantId", // Attribute to scan for
        secondaryKeyName = "customerId" // Secondary attribute filter (optional)
    )
)

val deletionKey = DynamoDbDeletionKeyValue(
    primaryKeyValue = "tenant-123",
    secondaryKeyValue = "customer-123" // Optional
)

DynamoDbDeletionConnector(dynamoDbClient).deleteData(deletionTarget, deletionKey)
```

### S3 deletion strategies

#### 1. Object Key Deletion (`OBJECT_KEY`)
Delete S3 objects by matching object key patterns.

```kotlin
val deletionTarget = S3DeletionTarget(
    strategy = S3DeletionStrategyType.OBJECT_KEY,
    awsRegion = "us-east-1",
    bucketName = "purchases",
    objectKeyPrefix = "data/customers/", // Optional S3 key prefix for efficient search
    deletionKeyPattern = Pattern.compile("data/customers/([\\w\\-]+)/.*") // Pattern with exactly one capture group
)

val deletionKey = S3DeletionKeyValue(
    deletionKeyPatternCaptureValue = "customer-123" // Matches capture group in pattern
)

S3DeletionConnector(s3Client).deleteData(deletionTarget, deletionKey)
```

#### 2. Row Level Deletion (`ROW_LEVEL`)
Remove specific rows from S3 files containing data from multiple customers.

```kotlin
val deletionTarget = S3DeletionTarget(
    strategy = S3DeletionStrategyType.ROW_LEVEL,
    awsRegion = "us-east-1",
    bucketName = "purchase-events",
    objectKeyPrefix = "data/events/", // Optional S3 key prefix for efficient search
    deletionKeyPattern = Pattern.compile("data/events/([\\w\\-]+)/.*"), // Pattern with capture group
    deletionRowAttributeName = "customerId", // Attribute to filter rows by
    objectFileFormat = FileFormat.JSONL // Supported: JSONL, PARQUET
)

val deletionKey = S3DeletionKeyValue(
    deletionKeyPatternCaptureValue = "customer-123", // Matches capture group in pattern
    deletionRowAttributeValue = "customer-123" // Value to match in row attribute
)

S3DeletionConnector(s3Client).deleteData(deletionTarget, deletionKey)
```

### Database Credentials Management

- Source credentials managed by each team and may change over time
- Database configuration includes references to credentials stored in other accounts
- Deletion platform CDK code allows deletion service to retrieve credentials from target account/secret
- No support for static credentials storage (security best practice)
- Credentials can be rotated by data store owner

## Technology Stack

- **Programming Language**: TypeScript for CDK, Kotlin for service code
- **Infrastructure**: AWS CDK
- **Compute**: AWS Lambda (with ECS Fargate migration path)
- **Storage**: Amazon DynamoDB
- **Messaging**: Amazon SQS
- **API**: AWS API Gateway (REST)
- **Authentication**: AWS IAM

## Software Packages

- **[aws-data-deletion-sdk](https://github.com/On-Demand-Deletion-Platform/aws-data-deletion-sdk)**: Kotlin library with backend logic for AWS data store deletion strategies
- **[aws-data-deletion-worker](https://github.com/On-Demand-Deletion-Platform/aws-data-deletion-worker)**: Service code for processing deletion requests and associated CDK infrastructure code
- **aws-data-deletion-target-api**: Deletion Target API service code and associated CDK infrastructure code

## Open Questions

- How to set up connectivity to DynamoDB tables inside private VPCs?
