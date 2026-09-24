# Azure App Service, not Container Apps

Chaptour is a single monolithic Razor Pages/MVC app with no services that need independent scaling — Container Apps' main advantages (per-service scaling, container portability) don't apply yet. App Service was chosen instead: no Dockerfile or container registry to maintain, and built-in staging-slot + swap gives zero-downtime deploys for free. Revisit if/when the app is actually split into independently-scaled services, or containerization is needed for another reason (none identified yet).
