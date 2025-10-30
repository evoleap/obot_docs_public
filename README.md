# obot

## Introduction

obot is a web application that orchestrates OLGA simulations across distributed compute. It provides a modern, containerized Rails 8 (Ruby 3) UI with Azure AD single sign-on, role-based administration, job scheduling and monitoring, rich file browsing/upload capabilities (local/SMB/remote adapters), and a token-secured API for automated submissions.

Key capabilities:

- Submit, queue, and monitor OLGA simulation jobs at scale
- Browse and upload case files from on-prem and remote shares with resilient adapters
- Administer users, roles, machines, and versions with performance-optimized views
- Integrate programmatically using scoped API tokens to submit and track jobs
- Operate in secure, private-networked environments with private endpoints and DNS


## Architecture at a glance

obot is delivered as multiple containers published to evoleap.azurecr.io and deployed into an Azure Container Apps environment:

- Web UI (Rails 8 + Ruby 3)
- File Browser Service
- File Manager
- Client Token Service
- Scheduler (Azure Functions container)
- Parallel Simulation Scaler (Azure Functions container)

Shared platform resources used by the deployment:

- Azure Database for MySQL Flexible Server (private endpoint)
- Azure Storage Account (Blob, Files, Queue, Table; private endpoint)
- Azure Virtual Network with dedicated subnets for database, storage, and the Container Apps Environment
- Private DNS zones for MySQL and Storage name resolution
- Application Insights + Log Analytics workspace for telemetry
- Managed identities for environment access (user-assigned/system-assigned)


## Deployment overview

This project is built and published as containers to evoleap.azurecr.io. Access to pull images is granted using scoped ACR tokens: we maintain an “obot” scope that includes all related containers (UI, File Manager, File Browser Service, Client Token Service, Scheduler Function, Parallel Sim Calculator Function). When a customer licenses obot, we issue an ACR token scoped to this “obot” scope; customers can then pull and deploy the images into their Azure Container Apps environment.

At a high level, an obot deployment includes:

1) Container Apps Environment (CAE)
	 - A managed environment for running the obot containers and Azure Functions containers.
	 - VNet integration with a dedicated subnet.

2) Database
	 - Azure Database for MySQL Flexible Server.
	 - Private endpoint and private DNS for secure connectivity.

3) Storage
	 - Azure Storage Account providing Blob/Files/Queue/Table services.
	 - Private endpoint and private DNS for name resolution inside the VNet.

4) Networking
	 - A Virtual Network with subnets for CAE, MySQL, and Storage.
	 - Private DNS Zones linked to the VNet for MySQL and Storage.

5) Observability & Identity
	 - Application Insights and a Log Analytics workspace.
	 - Managed identities for accessing platform resources as needed.

Customer deployment flow (conceptual):

- Obtain an ACR scope-limited pull token for the “obot” scope.
- Create or select a Container Apps Environment integrated with your VNet.
- Provision MySQL Flexible Server and a Storage Account with private endpoints into the VNet; link corresponding Private DNS zones.
- Deploy the obot containers from evoleap.azurecr.io into the CAE and configure environment variables/secrets (database, storage, SSO, API, etc.).
- Optionally enable Application Insights/Log Analytics and assign managed identities if required by your environment.

Note: The attached Bicep template in this repository demonstrates these resources and relationships at production scale. It is a useful reference for the required building blocks without prescribing every environment-specific setting.


## Configuration quick reference

- Authentication: Azure AD SSO via OmniAuth; configurable in the admin UI and application settings. Classic login is disabled once SSO is enabled.
- Database: Application connects to Azure Database for MySQL Flexible Server (Rails 8/Ruby 3 compatible); ensure private endpoint and credentials are configured.
- Storage: Uses Azure Storage (Blob/File/Queue/Table). Endpoints are typically accessed privately via the VNet.
- Secrets: Secrets are managed using Rails credentials; ensure your environments provide the appropriate master key/secret provisioning.
- API Tokens: Admins can create scoped API tokens to submit and monitor jobs programmatically; usage is tracked and visible in the admin UI.


## Product changes in the last 12 months

Highlights:

- Platform upgrade to Rails 8 and Ruby 3, with compatibility fixes across controllers, initializers, and reporting.
- Azure AD SSO added with admin-configurable settings; classic login disabled post-SSO.
- API Tokens introduced for job submission/monitoring with scopes, usage tracking, and UI management; Jobs API docs expanded with auxiliary files.
- Admin and jobs pages optimized and restyled; Inter font adopted; numerous UI/UX refinements and bug fixes.
- File browser subsystem refactored with adapter pattern; per-client upload support; improved reliability and special-character handling.
- Licensing hardened for containers (server GUID), and license usage charts fixed.
- Dockerfiles updated for newer Ruby; combined CA certificates added; Azure Pipeline/service connection fixes; secrets moved to Rails credentials.
- Database migrations for SSO, API tokens, licensing, and client upload flags.

By area:

- Platform and framework
	- Upgrade to Rails 8 and Ruby 3; new framework defaults; YAML permitted classes initializer.
	- Reports updated for Rails 8 (not backward compatible on older Rails).

- Authentication and security
	- Azure AD SSO via OmniAuth; admin UI for settings; linking existing users to SSO; disabled classic login post-SSO.

- API and integrations
	- API Tokens (models, scopes, usage entries, job associations) with admin UI and updated routes.
	- Jobs API documentation extended with auxiliary files.

- Jobs and scheduling UX
	- Added “limit” column, clickable job IDs, improved time formatting and listing behaviors.
	- Admin page performance improvements and better filtering.

- File browsing, storage, and uploads
	- Refactor to adapters/handlers for local/SMB/remote; increased timeouts and path/search fixes.
	- Per-client upload toggle backed by migrations and UI.

- Licensing
	- Container-friendly license key handling using a persistent server GUID.
	- License usage chart fixes.

- UX and styling
	- Inter font adoption; dropdown restyling; scrollbar and button fixes; favicon and width/layout tweaks; clearer warnings for unavailable versions.

- Infrastructure and configuration
	- Dockerfile updates, CA certificate bundling, Azure Pipelines service connection fixes.
	- Secrets moved to Rails credentials; environment defaults cleaned up.

- Database and migrations
	- New tables/columns for SSO (application settings, external login flags), API tokens/scopes/usage, server GUID for license keys, and upload-files flags.


## Breaking and operational notes

- Rails 8 + Ruby 3 runtime is required going forward.
- Reports updated for Rails 8 are not backward compatible.
- Enabling Azure AD SSO disables classic login paths; ensure Azure app registration and credentials are configured before switching.
- Run database migrations during upgrades to apply schema changes for SSO, API tokens, and licensing.
- Ensure Rails credentials/master key are provisioned in each environment.


## Related docs

- Jobs API (OpenAPI): `doc/obot-jobs-api.json`
- Jobs API auxiliary files: `doc/API_AUXILIARY_FILES.md`
- Refactoring summary (file browser adapters and related changes): `REFACTORING_SUMMARY.md`
