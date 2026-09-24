# .NET Aspire for local development orchestration

Chaptour is a single ASP.NET Core web app in production, not a distributed system — Aspire's usual pitch (multi-service orchestration) doesn't fully apply. We're using it anyway, scoped to local development: it orchestrates the local SQL Server and Azurite (Blob Storage emulator) containers, injects their connection strings automatically, and gives a dashboard for logs/telemetry while developing — replacing manual appsettings juggling or a hand-rolled docker-compose setup. Targets .NET 10.
