---
name: use-shared-postgresql
description: Add or connect a project-specific PostgreSQL database on the existing shared PostgreSQL flexible server. Use when a project needs PostgreSQL persistence, a new database, Aspire PostgreSQL configuration, Azure Bicep database provisioning, shared connection-string configuration, or CI/CD variables and secrets for shared PostgreSQL.
---

# Use shared PostgreSQL

## Apply the ownership rule

Treat PostgreSQL as shared infrastructure:

- Create only a project-specific database on the existing shared server.
- Never provision a project-specific PostgreSQL server.
- Reuse `SHARED_POSTGRES_SERVER_NAME` to provision the database.
- Reuse `SHARED_POSTGRES_CONNECTION_STRING` for host, port, credentials, SSL, and other server connection settings.
- Configure the project database name separately.
- Never introduce a database-specific connection-string secret.
- Override only the `Database` component of the shared connection string at runtime.

Use lowercase database names containing only letters, digits, and underscores unless the repository already defines another convention.

## Inspect before changing

1. Read the repository instructions and infrastructure conventions.
2. Find existing shared PostgreSQL parameters, secrets, modules, and naming.
3. Identify every runtime that needs the database, such as API, MCP, workers, and migration jobs.
4. Check whether the repository uses Aspire, Bicep, Terraform, EF Core, Npgsql, or another database client.
5. Preserve existing shared-server resources and unrelated databases.

## Implement the database

1. Define a database-name setting with a clear project-specific key.
2. Add a database resource as a child of the existing shared PostgreSQL server.
3. Pass the shared connection string and database name independently to every consuming runtime.
4. Build the effective runtime connection string by parsing the shared connection string and replacing only its database property.
5. For local Aspire development, add the project database to the local PostgreSQL resource and expose it under the same shared-connection configuration key used in deployment.
6. Add migrations or deterministic schema initialization according to repository conventions.

Read [implementation-patterns.md](references/implementation-patterns.md) for .NET, Aspire, Bicep, and GitHub Actions patterns.

## Keep configuration aligned

Use these portable names unless the repository already has canonical equivalents:

- Variable: `SHARED_POSTGRES_SERVER_NAME`
- Secret: `SHARED_POSTGRES_CONNECTION_STRING`
- Project setting: `<ProjectName>Database:Name`
- Default database name: project slug in lowercase

The shared connection string may already contain a database. Do not trust or mutate that source value globally. Override it only in the project-owned client configuration.

## Validate

Before completing the change:

1. Search for dedicated PostgreSQL server resources introduced by the change and remove them.
2. Search for project-specific connection-string secrets and replace them with the shared secret.
3. Confirm the database resource is parented to the shared PostgreSQL server.
4. Confirm all consuming runtimes receive both shared connection information and the database name.
5. Confirm the effective connection string selects the project database.
6. Compile infrastructure and synchronize generated artifacts.
7. Build the full solution and run relevant tests.
8. Update repository instructions so future work preserves the shared-server rule.

Report clearly that creating the database resource does not create a new PostgreSQL server.
