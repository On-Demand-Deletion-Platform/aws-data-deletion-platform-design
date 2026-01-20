# Technical Design Options Analysis - On Demand Deletion Platform

## Deletion Processor Architecture Options

### Queue-Based Service Architecture

**Description**: Queue-based service that accepts customer-level deletion requests, maps each customer deletion request to multiple table-level deletion strategies, and orchestrates table-level deletion requests.

**Pros**:
- Asynchronous processing suitable for sparse customer-level deletion requests
- Fault tolerance with automatic retries
- Cost efficiency at expected scale
- Lower implementation effort than ECS/EC2
- Scales to wide fan-out of backend deletion requests

**Cons**:
- Limited to Lambda execution time constraints
- May need migration to ECS Fargate for long-running compute

**Recommendation**: Start with Lambda for simplicity and low cost, migrate to ECS Fargate if compute time exceeds ~7 hours/day for more cost-effective long-running compute.

### Technology Choices

#### Client-Facing Interface: SQS Queue vs API Gateway

**SQS Queue**:
- **Pros**: Good fit for asynchronous deletion requests, critical eventual processing, low-cost ($0.40/million queue requests/month), low implementation effort
- **Cons**: Not suitable for synchronous responses

**API Gateway**:
- **Pros**: Synchronous responses, built-in authentication/authorization
- **Cons**: Higher cost for high-volume requests, more complex for asynchronous processing

**Recommendation**: SQS queue for customer-level deletion requests due to acceptable asynchronous completion and cost efficiency.

#### Compute Service: Lambda vs ECS/EC2

**Lambda**:
- **Pros**: Simplest option, lowest cost for sparse requests, within free tier limits, fast processing for customer-to-table-level conversion
- **Cons**: Execution time limits, cost scaling at high compute volumes

**ECS/EC2**:
- **Pros**: No execution time limits, lower average/p90 latency for long-lived service, deduplicate deletion target look-ups, cost-effective for sustained high compute
- **Cons**: Higher implementation complexity, higher baseline costs

**Recommendation**: Start with Lambda, migrate when compute time exceeds ~7 hours/day threshold.

#### Task Queue: SQS vs DynamoDB

**SQS**:
- **Pros**:
  - ~40% lower cost than DynamoDB for use case
  - 1-2 weeks lower integration effort
  - Built-in retry mechanisms
  - Prevents duplication of successful backend deletions
- **Cost Analysis**: $0.0000008 per million table-level deletion requests vs $0.0000013125 for DynamoDB

**DynamoDB**:
- **Pros**: More flexible querying patterns, can structure and index data to group by AWS account ID + technology to optimize backend API requests
- **Cons**: Higher cost, higher implementation effort

**Recommendation**: SQS for lower cost and faster implementation.

## Deletion Target API Architecture Options

### Database: DynamoDB vs SQL Databases

**DynamoDB**:
- **Pros**:
  - Good fit for CRUD use cases
  - Significantly lower cost (expect <$1/month)
  - Higher availability than SQL databases
  - Lower maintenance overhead
- **Cons**: Limited querying flexibility, more expensive to restructure data if needed to support future use cases

**SQL Databases (RDS)**:
- **Pros**: Rich querying capabilities, ACID compliance
- **Cons**: Higher cost, more maintenance, lower availability

**Recommendation**: DynamoDB for cost efficiency and operational simplicity.

### API Interface: REST API Gateway vs HTTP API Gateway vs Load Balancer

**REST API Gateway**:
- **Pros**:
  - Native support for authentication/authorization with resource-level access controls
  - Built-in metrics, rate limiting, throttling
  - API-gateway-level caching
  - SDK generation
  - Execution logs and method-level metrics
  - Request validation
- **Cons**: Slightly higher cost than HTTP API Gateway

**HTTP API Gateway**:
- **Pros**: Lower cost
- **Cons**: Fewer built-in features

**Load Balancer + Lambda**:
- **Pros**: Maximum flexibility
- **Cons**: Higher implementation and maintenance effort

**Recommendation**: REST API Gateway for reduced implementation effort and built-in functionality.

### Authentication & Authorization: IAM vs Auth Lambda

**IAM Auth**:
- **Pros**:
  - Native AWS support
  - Low implementation/maintenance effort
  - Suitable for AWS-hosted service consumers
  - Fine-grained permissions (read-only for deletion workers, read/write for management services)
- **Cons**: Limited to AWS ecosystem

**Custom Authentication Lambda**:
- **Pros**: More flexibility, supports granular row-level authorization logic
- **Cons**: Higher implementation complexity, not required for current use cases

**Recommendation**: IAM Auth for single-tenant AWS-native architecture.

## Programming Language Options

### CDK: TypeScript vs Other Languages

**TypeScript**:
- **Pros**: Best-supported CDK language, most documentation, most production examples
- **Cons**: None significant for CDK use case

**Recommendation**: TypeScript for CDK implementation.

### Service Code: Kotlin vs Java vs TypeScript vs Rust

**Kotlin**:
- **Pros**:
  - JVM ecosystem with strong AWS support
  - Low-effort database integrations via JDBC
  - Less boilerplate than Java
  - Fully interoperable with Java
  - Faster development cycles
- **Cons**: Smaller community than Java

**Java**:
- **Pros**: Mature AWS SDK, extensive library ecosystem
- **Cons**: More boilerplate than Kotlin

**TypeScript**:
- **Pros**: Consistent with CDK choice
- **Cons**: Less mature database connector libraries compared to JVM

**Rust**:
- **Pros**: Superior performance
- **Cons**: Limited AWS SDK support, fewer examples, performance not critical (database queries are bottleneck)

**Recommendation**: Kotlin for reduced boilerplate while maintaining JVM ecosystem benefits and mature AWS/database support.
