# Identity and Access Management

!!! tip "Running alongside ontul?"
    In a federated deployment, **ontul** is the IAM authority and NeorunBase pulls its users, groups, and
    policies (read-only) to enforce them for clients connecting directly to NeorunBase. See
    [IAM Federation (ontul)](iam-federation.md).

NeorunBase includes a built-in Identity and Access Management (IAM) system that provides authentication, authorization, and fine-grained access control.

## User Management

NeorunBase supports creating and managing database users with:

- Username and password-based authentication, with passwords stored as PBKDF2-HMAC-SHA256 hashes
- Access key and secret key credentials for programmatic access
- Temporary credentials with configurable expiration
- Federated identities from an external provider, which carry groups from your directory
  rather than having an account here — see [Single Sign-On](sso.md)

## Groups

Users can be organized into IAM groups. Policies attached to a group apply to all members, simplifying access management for teams and applications.

## Policies

NeorunBase uses policy-based access control to manage permissions:

- Policies define what actions are allowed or denied on specific resources (catalogs, schemas, tables).
- Policies can be attached to users or groups.
- Multiple policies can be combined, with deny rules taking precedence over allow rules.
- Fine-grained controls include column-level allow/deny, **column masking** (return a SQL expression instead of the raw value), and row-level filters.
- Catalog management (`CREATE` / `ALTER` / `DROP CATALOG`) is gated by catalog-level resources (`db:catalog:<name>`).

## Admin API

User, group, and policy management is available through the NeorunBase Admin API, allowing administrators to manage access control programmatically or through the [Admin UI](admin-ui.md#iam).

The admin API is for **administrators** (a policy allowing `*` on `*`, e.g. through `admin-group`). A signed-in user who is not an administrator can read only the cluster view (`/admin/nodes/*`, `/admin/topology`, `/admin/zookeeper`, `/admin/monitoring/metrics`); every other endpoint — including all IAM endpoints and the REST SQL endpoints `/admin/query/execute` and `/v1/api/*`, which do not go through pg-wire's per-user policy evaluation — answers **403**. Non-administrators work with data over pg-wire, where their policies, column masks and row filters apply. See [Admin UI → Who Can Use What](admin-ui.md#who-can-use-what).

Access keys and STS temporary credentials work on pg-wire: an access key signs in as user name with its secret as the password; temporary credentials sign in with the temporary access key as user name and `<secret>::<session token>` as the password. Temporary credentials replicate to every coordinator, so they work whichever coordinator a client reaches.

For the full policy JSON schema, resource format, action catalog, column- and row-level ACL, and worked examples, see the [IAM Policy Reference](iam-policy.md).
