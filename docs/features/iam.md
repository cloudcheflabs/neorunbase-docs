# Identity and Access Management

NeorunBase includes a built-in Identity and Access Management (IAM) system that provides authentication, authorization, and fine-grained access control.

## User Management

NeorunBase supports creating and managing database users with:

- Username and password-based authentication
- Access key and secret key credentials for programmatic access
- Temporary credentials with configurable expiration

## Groups

Users can be organized into IAM groups. Policies attached to a group apply to all members, simplifying access management for teams and applications.

## Policies

NeorunBase uses policy-based access control to manage permissions:

- Policies define what actions are allowed or denied on specific resources (schemas, tables).
- Policies can be attached to users or groups.
- Multiple policies can be combined, with deny rules taking precedence over allow rules.

## Admin API

User, group, and policy management is available through the NeorunBase Admin API, allowing administrators to manage access control programmatically or through the Admin UI.
