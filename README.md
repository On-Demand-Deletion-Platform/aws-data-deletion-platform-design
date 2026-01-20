# aws-data-deletion-platform-design
Design artifacts for the AWS On Demand Data Deletion Platform.

## AWS Deletion Platform system architecture
![System architecture diagram for the AWS On Demand Data Deletion Platform](/diagrams/On%20Demand%20Data%20Deletion%20Platform.png)

### Deletion Processor
**Description**
* Queue-based service that processes customer data deletion requests, fanning out to backend data-store-level deletion requests.
* Backend target data stores may be distributed across many AWS accounts and technologies.

**Technical Decisions**
* **SQS + Lambda architecture**: Chosen for asynchronous processing, fault tolerance, and cost efficiency at expected scale. SQS provides reliable queuing with automatic retries, while Lambda offers serverless compute within free tier limits.
* **Two-stage processing**: Customer requests are first processed to generate multiple backend deletion tasks, then each backend task is processed independently. This separation enables fan-out to multiple data stores and prevents duplicate work on partial failures.
* **SQS over DynamoDB for task queue**: Selected for ~40% lower cost and faster implementation, with sufficient durability guarantees for the use case.
* **Lambda scaling strategy**: Start with Lambda for simplicity and low cost, migrate to ECS Fargate if compute time exceeds ~7 hours/day for more cost-effective long-running compute.
* **Authentication & Authorization: IAM** - Use AWS IAM roles and policies to minimize resource access to authorized service components.

### Deletion Target API
**Description**
* CRUD API for onboarding, offboarding, enabling, and disabling deletion targets.
* Stores deletion targets in a DeletionTargets DynamoDB table
  * Partition key: `"ACCT#<awsAccountId>#TECH#<technology>"` (eg. `"ACCT#123456789012#TECH#dynamodb"`)
  * Sort key: deletion target UUID

**Use cases, with expected transactions per month**
* Create a deletion target: < 1000 transactions/month
* Update a deletion target: < 1000 transactions/month
* Delete a deletion target: < 10 transactions/month
* List all deletion targets for a given AWS account + technology: < 1000 transactions/month
* Query all deletion targets, grouped by AWS account + technology: < 1000 transactions/month

**Technical Decisions**
* **Database: AWS DynamoDB** - Good fit for use cases, significantly lower cost than other options (expect < $1/month for foreseeable future), higher availability and lower maintenance than SQL-based databases
* **Compute service: AWS Lambda** - Simplest and least expensive option for expected scale, expect to remain within free tier
* **API interface: AWS REST API Gateway** - Provides native support for authentication, authorization, metrics, rate limiting, and request validation with minimal implementation effort
* **Authentication & Authorization: IAM** - Use AWS IAM roles and policies to grant deletion workers read-only access, management services read/write access.

## License
The resources in this project are released under the [GPL-3.0 License](LICENSE).
