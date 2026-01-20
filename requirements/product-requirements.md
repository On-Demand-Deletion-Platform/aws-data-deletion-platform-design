# Product Requirements Document - On Demand Deletion Platform

## Introduction

The On Demand Deletion Platform is an open-source service for orchestrating customer data deletion requests across heterogeneous data stores, in support of privacy regulations that require the right to erasure (e.g., GDPR).

Implementing on-demand deletion correctly is non-trivial. Engineering teams must identify all data stores that contain customer data, understand each store's schema and deletion semantics, and build reliable mechanisms to delete or anonymize records in response to a request. In practice, this often spans multiple technologies, such as MySQL, PostgreSQL, DynamoDB, Redshift, and S3, each with different access patterns, credentials, network boundaries, and data models. As a result, teams frequently spend weeks to months designing, implementing, and validating on-demand deletion logic.

The On Demand Deletion Platform standardizes this work by providing built-in connectors and configuration templates for common data store technologies. Data stores are onboarded using declarative configuration and AWS CDK constructs, enabling teams to define connection details, deletion behavior, and validation checks without writing custom orchestration code. This allows engineers to deploy and verify on-demand deletion support for a new data store in minutes rather than weeks.

At scale, this approach significantly reduces engineering effort. For example, a company with approximately 1,000 data stores subject to on-demand deletion requirements would save on the order of 2,000 engineer-weeks (~40 engineer-years) by using our platform instead of duplicating effort across teams to design, implement, and validate custom deletion mechanisms for each data store.

## Business Requirements

### P0 Business Requirements

1. **Multi-Technology Data Store Support**: As a client, I must be able to define and deploy on-demand-deletion configurations for any data stores I own hosted in the following services:
   - AWS DynamoDB
   - AWS Aurora PostgreSQL
   - AWS Aurora MySQL
   - AWS RDS PostgreSQL
   - AWS RDS MySQL
   - AWS S3

2. **Rapid Onboarding**: As a client, I should require under a day of engineering effort to onboard each incremental data store to on-demand deletion.
   - Target 1 hour of effort for permissions/networking config, defining and deploying each incremental data store deletion configuration, given the client has already deployed the deletion platform and has a pre-existing data store deletion configuration.
   - Target 1 day of effort to deploy the initial deletion platform and its first data store deletion configuration, assuming no prior knowledge of the platform.

### P1 Business Requirements

1. **Configuration Management**: As a client, I want to be able to enable/disable a given data store's deletion configuration.

2. **User Interface**: As a client, I want to be able to easily view and edit all data store configurations through a UI.

3. **Status Visibility**: As a client, I want to be able to view all my data stores' deletion enabled statuses through a UI.

4. **Validation**: As a client, I want to be able to automatically validate correctness of on-demand-deletion configurations through standardized mechanisms.
   - Example: Submit a test request for a non-existent customer and validate the database connection is successful and get the expected record not found error or a successful response.

5. **Extended Technology Support**: As a client, I want to be able to onboard data stores hosted in the following services:
   - AWS Redshift
   - AWS DocumentDB
   - AWS Neptune
   - AWS RDS SQL Server
   - AWS RDS MariaDB
   - Azure SQL Database
   - Azure Cosmos DB (commonly used for Azure-based non-relational data)
