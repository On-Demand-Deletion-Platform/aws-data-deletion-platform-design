# Technical Requirements - On Demand Deletion Platform

## Technical Requirements

### P0 Technical Requirements

1. **Extensibility**: The platform must be extensible to multiple data store technologies (e.g., DynamoDB, Redshift, RDS MySQL, RDS PostgreSQL, etc).

2. **DynamoDB Support**: The platform must support DynamoDB tables at MVP launch.

3. **Retry Capability**: The platform must allow redriving failed data deletion requests.

4. **Single-Tenant Architecture**: The platform must be single-tenant, so that each instance of the platform is specific to a single business and can only give that business its own internal data store metadata.

## Security Requirements

### Availability

1. **High Availability SLA**: The logical component accepting customer deletion requests must have a monthly availability SLA of no less than 99.9%.
   - Clarification: It's acceptable for components executing the actual data deletion to be offline for up to a few days, but we must minimize loss of incoming/unprocessed deletion requests.

2. **Retry Mechanisms**: The platform must have built-in retry mechanisms that enforce that data deletion requests are eventually processed even if backend services are temporarily unavailable.

### Confidentiality

1. **No Customer Data Storage**: The platform must not itself store any customer personal data. It should only store data store metadata required to orchestrate deletion requests.

2. **Access Control**: No unauthorized user or service should be able to view/access any client-specific data store configurations or data deletion requests.

### Integrity

1. **Tenant Isolation**: On Demand Deletion Platform instances must be strictly isolated between tenants, with tenant-specific permissions, to enforce that data owned by a given client can only be accessed/deleted by that client.

2. **Audit Trail**: The platform must generate an audit log of all accepted data deletion requests and all data deletion commands that have been executed.
