# EF Core migrations apply automatically as a pipeline step

Pending migrations are applied to Azure SQL as their own step in the deploy pipeline (not at app startup, which would race across multiple App Service instances), with no manual approval gate. This is a deliberate, revisit-later choice: there's no real user data at stake yet, so the risk a manual gate protects against doesn't exist yet either. Once the database holds data worth protecting, add a manual review/approval step before migrations apply in production — don't carry this automation forward by default.
