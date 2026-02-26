---
name: azure-deploy
description: Deploy applications to Azure App Service, Azure Functions, Static Web Apps, and API Management (APIM). USE THIS SKILL when users want to deploy, publish, host, or run their application on Azure, or deploy APIM for API Gateway/AI Gateway scenarios. This skill detects application type (React, Vue, Angular, Next.js, Python, .NET, Java, etc.), recommends the optimal Azure service, provides local preview capabilities, and guides deployment. Trigger phrases include "deploy to Azure", "host on Azure", "publish to Azure", "run on Azure", "get this running in the cloud", "deploy my app", "Azure deployment", "set up Azure hosting", "deploy to App Service", "deploy to Functions", "deploy to Static Web Apps", "deploy APIM", "create API Management", "deploy API gateway", "preview locally", "test before deploying", "what Azure service should I use", "help me deploy", etc. Also handles multi-service deployments with Azure Developer CLI (azd) and Infrastructure as Code when complexity is detected.
---

## Preferred: Use azd for Deployments

> **Prefer `azd` (Azure Developer CLI) over raw `az` CLI for deployments.**
> Use `az` CLI for resource queries, simple single-resource deployments, or when explicitly requested.

**Why azd is preferred:**
- **Faster** - provisions resources in parallel
- **Automatic ACR integration** - no manual credential setup
- **Single command** - `azd up` does everything
- **Infrastructure as Code** - reproducible deployments with Bicep
- **Environment management** - easy dev/staging/prod separation

**When `az` CLI is acceptable:**
- Single-resource deployments without IaC requirements
- Quick prototyping or one-off deployments
- User explicitly requests `az` CLI
- Querying or inspecting existing resources

> ⚠️ **IMPORTANT: For automation and agent scenarios**, always use the `--no-prompt` flag with azd commands to prevent interactive prompts from blocking execution:
> ```bash
> azd up --no-prompt
> azd provision --no-prompt
> azd deploy --no-prompt
> ```

## MCP Tools for azd Workflows

Use the Azure MCP server's azd tools (`azure-azd`) for validation and guidance:

| Command | Description |
|---------|-------------|
| `validate_azure_yaml` | **Validates azure.yaml against official JSON schema** - Use before deployment |
| `discovery_analysis` | Analyze application components for AZD migration |
| `architecture_planning` | Select Azure services for discovered components |
| `azure_yaml_generation` | Instructions for generating azure.yaml |
| `docker_generation` | Generate Dockerfiles for Container Apps/AKS |
| `infrastructure_generation` | Generate Bicep templates |
| `iac_generation_rules` | Get Bicep compliance rules and best practices |
| `project_validation` | Comprehensive validation before deployment |
| `error_troubleshooting` | Diagnose and troubleshoot azd errors |

**Always validate azure.yaml before deployment:**
```javascript
// Validate azure.yaml using MCP tool
const validation = await azure-azd({
  command: "validate_azure_yaml",
  parameters: { path: "./azure.yaml" }
});
```

### Additional MCP Tools

For deployment planning and logs, use `azure__deploy`:

| Tool | Command | Description |
|------|---------|-------------|
| `azure__deploy` | `deploy_plan_get` | Generate a deployment plan for Azure infrastructure and applications |
| `azure__deploy` | `deploy_iac_rules_get` | Get IaC (Bicep/Terraform) guidelines |
| `azure__deploy` | `deploy_app_logs_get` | Fetch logs from deployed apps (Container Apps, App Service, Functions) |
| `azure__deploy` | `deploy_pipeline_guidance_get` | Get CI/CD pipeline guidance |
| `azure__deploy` | `deploy_architecture_diagram_generate` | Generate Azure service architecture diagrams |

## Capabilities

Deploy applications to Azure with intelligent service selection, local preview, and guided deployment workflows.

---

## Quick Start Decision Tree

```
User wants to deploy → Run detection workflow below
```

---

## Phase 1: Application Detection

**ALWAYS start by scanning the user's project to detect the application type.**

### Step 1.1: Check for Existing Azure Configuration

Look for these files first (HIGH confidence signals):

| File Found | Recommendation | Action |
|------------|----------------|--------|
| `azure.yaml` | Already configured for azd | **Validate first**, then use `azd up` to deploy |
| `function.json` or `host.json` | Azure Functions project | **See [Azure Functions Guide](./reference/functions.md)** |
| `staticwebapp.config.json` or `swa-cli.config.json` | Static Web Apps project | **See [Static Web Apps Guide](./reference/static-web-apps.md)** |

**Check for API Management / AI Gateway requests:**

| User Request | Recommendation | Action |
|--------------|----------------|--------|
| "Deploy APIM", "create API Management", "API gateway" | Deploy API Management | **See [APIM Deployment Guide](../../azure-prepare/references/apim.md)** |
| "AI gateway", "configure AI model", "add Azure OpenAI backend" | AI Gateway configuration | Deploy with **[APIM Guide](../../azure-prepare/references/apim.md)**, configure with **azure-aigateway skill** |

> 💡 **When to use API Management:**
> - User wants to expose APIs with policies (rate limiting, auth, caching)
> - User needs an AI Gateway for Azure OpenAI or AI Foundry models
> - User mentions APIM, API Management, or API gateway
> 
> **📖 See [APIM Deployment Guide](../../azure-prepare/references/apim.md)** for deployment, then use **azure-aigateway skill** for AI-specific configuration.

**When `azure.yaml` is found, validate before deployment:**
```javascript
// Validate azure.yaml using MCP tool before proceeding
const validation = await azure-azd({
  command: "validate_azure_yaml",
  parameters: { path: "./azure.yaml" }
});

if (validation.errors && validation.errors.length > 0) {
  // Fix validation errors before deployment
  console.error("azure.yaml validation failed:", validation.errors);
} else {
  // Proceed with azd up
}
```

If found and valid, route to the appropriate specialized skill.

> 💡 **When to use Static Web Apps:**
> - Project has `staticwebapp.config.json` or `swa-cli.config.json`
> - User wants to deploy static frontends (React, Vue, Angular, etc.) to Azure
> - User needs local development emulation with SWA CLI
> - User wants to add Azure Functions APIs to their static site
> - User mentions Static Web Apps or SWA CLI
> 
> **📖 See [Static Web Apps Deployment Guide](./reference/static-web-apps.md)** for specialized guidance on SWA CLI configuration, local emulation, GitHub Actions workflows, and database connections.

**Check for containerization signals:**

| File/Indicator Found | Recommendation | Action |
|---------------------|----------------|--------|
| `Dockerfile` | Containerized application | **See [Container Apps Guide](./reference/container-apps.md)** for deployment |
| `docker-compose.yml` | Multi-container application | **See [Container Apps Guide](./reference/container-apps.md)** for deployment |
| User mentions "container", "Docker", "scheduled task", "cron job", "batch processing" | Container-based deployment | **See [Container Apps Guide](./reference/container-apps.md)** |

> 💡 **When to use Container Apps:**
> - Application is already containerized (has Dockerfile)
> - User wants to deploy multiple containers together
> - User needs scheduled tasks, cron jobs, or event-driven batch processing
> - User mentions Container Apps or wants serverless containers
> 
> **📖 See [Container Apps Deployment Guide](./reference/container-apps.md)** for specialized guidance on Docker validation, ACR integration, Container Apps Jobs, and multi-container orchestration.

### Step 1.2: Detect Application Framework

Scan for configuration files and dependencies:

**Node.js / JavaScript / TypeScript:**
```
package.json exists →
├── next.config.js/mjs/ts → Next.js
│   ├── Has `output: 'export'` → Static Web Apps (SSG)
│   └── Has API routes or no export config → App Service (SSR)
├── nuxt.config.ts/js → Nuxt
│   ├── Has `ssr: false` or `target: 'static'` → Static Web Apps
│   └── Otherwise → App Service (SSR)
├── angular.json → Angular → Static Web Apps
│   └── **See [Static Web Apps Guide](./reference/static-web-apps.md)**
├── vite.config.* → Vite-based (React/Vue/Svelte) → Static Web Apps
│   └── **See [Static Web Apps Guide](./reference/static-web-apps.md)**
├── gatsby-config.js → Gatsby → Static Web Apps
│   └── **See [Static Web Apps Guide](./reference/static-web-apps.md)**
├── astro.config.mjs → Astro → Static Web Apps
│   └── **See [Static Web Apps Guide](./reference/static-web-apps.md)**
├── nest-cli.json → NestJS → App Service
├── Has express/fastify/koa/hapi dependency → App Service
└── No framework, just static build → Static Web Apps
```

**Python:**
```
requirements.txt or pyproject.toml exists →
├── function_app.py exists → Azure Functions (v2 programming model)
│   └── **See [Azure Functions Guide](./reference/functions.md)**
├── Has flask dependency → App Service
├── Has django dependency → App Service
├── Has fastapi dependency → App Service
└── Has azure-functions dependency → Azure Functions
    └── **See [Azure Functions Guide](./reference/functions.md)**
```

> 💡 **When to use Azure Functions:**
> - Project has `host.json`, `local.settings.json`, or `function_app.py`
> - User wants serverless APIs, event-driven functions, or timer-triggered jobs
> - User mentions Azure Functions, triggers, bindings, or webhooks
> 
> **📖 See [Azure Functions Deployment Guide](./reference/functions.md)** for specialized guidance on function initialization, trigger configuration, deployment slots, and function-specific troubleshooting.

**.NET:**
```
*.csproj or *.sln exists →
├── <AzureFunctionsVersion> in csproj → Azure Functions
│   └── **See [Azure Functions Guide](./reference/functions.md)**
├── Blazor WebAssembly project → Static Web Apps
├── ASP.NET Core web app → App Service
└── .NET API project → App Service
```

**Java:**
```
pom.xml or build.gradle exists →
├── Has azure-functions-* dependency → Azure Functions
│   └── **See [Azure Functions Guide](./reference/functions.md)**
├── Has spring-boot dependency → App Service
└── Standard web app → App Service
```

**Static Only:**
```
index.html exists + no package.json/requirements.txt →
└── Pure static site → Static Web Apps
    └── **See [Static Web Apps Guide](./reference/static-web-apps.md)**
```

### Step 1.3: Detect Multi-Service Architecture

Check for complexity indicators that suggest azd + IaC:

```
Multi-service triggers:
├── Monorepo structure (frontend/, backend/, api/, packages/, apps/)
├── docker-compose.yml with multiple services
├── Multiple package.json in different subdirectories
├── Database references in config (connection strings, .env files)
├── References to Redis, Service Bus, Event Hubs, Storage queues
├── User mentions "multiple environments", "staging", "production"
└── More than one deployable component detected
```

**If multi-service detected → Recommend azd + Infrastructure as Code**
See [Multi-Service Deployment Guide](./reference/multi-service.md)

### Step 1.4: Confidence Assessment

After detection, assess confidence:

| Confidence | Criteria | Action |
|------------|----------|--------|
| **HIGH** | Azure config file found (azure.yaml, function.json, staticwebapp.config.json) | Proceed with detected service |
| **MEDIUM** | Framework detected from dependencies | Explain recommendation, ask for confirmation |
| **LOW** | Ambiguous or no clear signals | Ask clarifying questions |

**Clarifying questions for LOW confidence:**
1. "What type of application is this? (static website, API, full-stack, serverless functions, containerized app)"
2. "Is your application already containerized with Docker?"
3. "Does your app need server-side rendering or is it purely client-side?"
4. "Do you need scheduled tasks, cron jobs, or event-driven processing?"
5. "Will you need a database, caching, or other Azure services?"

---

## Phase 2: Local Preview (No Azure Auth Required)

Before deploying, help users test locally.

### Static Web Apps - Local Preview
```bash
# Install SWA CLI (one-time)
npm install -g @azure/static-web-apps-cli

# Start local emulator
swa start

# Or with specific paths
swa start ./dist --api-location ./api

# With a dev server proxy
swa start http://localhost:3000 --api-location ./api
```

### Azure Functions - Local Preview
```bash
# Install Azure Functions Core Tools (one-time)
npm install -g azure-functions-core-tools@4

# Start local Functions runtime
func start

# With specific port
func start --port 7071
```

### App Service Apps - Local Preview
Use the framework's built-in dev server:

```bash
# Node.js
npm run dev
# or
npm start

# Python Flask
flask run

# Python FastAPI
uvicorn main:app --reload

# .NET
dotnet run

# Java Spring Boot
./mvnw spring-boot:run
```

See [Local Preview Guide](./reference/local-preview.md) for detailed setup instructions.

---

## Phase 3: Prerequisites & Dependency Management

**ALWAYS check and install missing dependencies before proceeding.**

### 3.1 Azure Authentication (Auto-Login)

**Check login status and automatically login if needed:**
```bash
# Check if logged in, auto-login if not
if ! az account show &>/dev/null; then
    echo "Not logged in to Azure. Starting login..."
    az login
fi

# Verify and show current subscription
az account show --query "{name:name, id:id}" -o table
```

If the user has multiple subscriptions, help them select the correct one:
```bash
# List all subscriptions
az account list --query "[].{Name:name, ID:id, Default:isDefault}" -o table

# Set subscription
az account set --subscription "<name-or-id>"
```

### 3.2 Dependency Detection & Auto-Install

Run this check first and install any missing tools:

```bash
# Check all dependencies at once
check_deps() {
  local missing=()
  command -v az &>/dev/null || missing+=("azure-cli")
  command -v func &>/dev/null || missing+=("azure-functions-core-tools")
  command -v swa &>/dev/null || missing+=("@azure/static-web-apps-cli")
  command -v azd &>/dev/null || missing+=("azd")
  echo "${missing[@]}"
}
```

### 3.3 Install Missing Tools

**Azure CLI** (required for all deployments):
```bash
# macOS
brew install azure-cli

# Windows (PowerShell)
winget install Microsoft.AzureCLI

# Linux (Ubuntu/Debian)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```
Multi-service triggers:
├── Monorepo structure (frontend/, backend/, api/, packages/, apps/)
├── docker-compose.yml with multiple services
├── Multiple package.json in different subdirectories
├── Database references in config (connection strings, .env files)
├── References to Redis, Service Bus, Event Hubs, Storage queues
├── User mentions "multiple environments", "staging", "production"
└── More than one deployable component detected
```

**If multi-service detected → Recommend azd + Infrastructure as Code**
See [Multi-Service Deployment Guide](./reference/multi-service.md)

### Step 1.4: Confidence Assessment

After detection, assess confidence:

| Confidence | Criteria | Action |
|------------|----------|--------|
| **HIGH** | Azure config file found (azure.yaml, function.json, staticwebapp.config.json) | Proceed with detected service |
| **MEDIUM** | Framework detected from dependencies | Explain recommendation, ask for confirmation |
| **LOW** | Ambiguous or no clear signals | Ask clarifying questions |

**Clarifying questions for LOW confidence:**
1. "What type of application is this? (static website, API, full-stack, serverless functions, containerized app)"
2. "Is your application already containerized with Docker?"
3. "Does your app need server-side rendering or is it purely client-side?"
4. "Do you need scheduled tasks, cron jobs, or event-driven processing?"
5. "Will you need a database, caching, or other Azure services?"

---

## Specialized Deployment Skills

**Azure Functions Core Tools** (for Functions projects):
```bash
# npm (all platforms)
npm install -g azure-functions-core-tools@4

# macOS
brew tap azure/functions && brew install azure-functions-core-tools@4

# Windows
winget install Microsoft.AzureFunctionsCoreTools
```

**Static Web Apps CLI** (for SWA projects):
```bash
npm install -g @azure/static-web-apps-cli
```

**Azure Developer CLI** (for multi-service/IaC):
```bash
# macOS
brew install azd

# Windows
winget install Microsoft.Azd

# Linux
curl -fsSL https://aka.ms/install-azd.sh | bash
```

### 3.4 Project Dependencies

Detect and install project-level dependencies:

```bash
# Node.js - install if node_modules missing
[ -f "package.json" ] && [ ! -d "node_modules" ] && npm install

# Python - create venv and install if missing  
[ -f "requirements.txt" ] && [ ! -d ".venv" ] && python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt

# .NET - restore packages
[ -f "*.csproj" ] && dotnet restore

# Java - install dependencies
[ -f "pom.xml" ] && mvn dependency:resolve
```

---

## Phase 4: Single-Service Deployment (Azure CLI)

### 4.1 Static Web Apps Deployment

> 💡 **For advanced Static Web Apps scenarios**, see the **[Static Web Apps Deployment Guide](./reference/static-web-apps.md)** which provides:
> - SWA CLI configuration and local emulation
> - Framework-specific setup guidance
> - Azure Functions API integration
> - GitHub Actions workflow automation
> - Database connections setup
> - Advanced routing and authentication configuration

**Create resource and deploy:**
```bash
# Create resource group (if needed)
az group create --name <resource-group> --location <location>

# Create Static Web App
az staticwebapp create \
  --name <app-name> \
  --resource-group <resource-group> \
  --location <location> \
  --sku Free

# Get deployment token
az staticwebapp secrets list \
  --name <app-name> \
  --resource-group <resource-group> \
  --query "properties.apiKey" -o tsv

# Deploy with SWA CLI
swa deploy ./dist \
  --deployment-token <token> \
  --env production
```

**Plain HTML Site (Single File or No Build Step):**

For plain HTML sites without a build process, SWA CLI requires content in a dedicated folder:
```bash
# Create output directory and copy files
mkdir -p dist && cp -r *.html *.css *.js *.png *.jpg *.svg dist/ 2>/dev/null || true

# Get deployment token
TOKEN=$(az staticwebapp secrets list \
  --name <app-name> \
  --resource-group <resource-group> \
  --query "properties.apiKey" -o tsv)

# Deploy from dist folder
swa deploy ./dist --deployment-token "$TOKEN" --env production

# Clean up temp folder (optional)
rm -rf dist
```

**Using azd for Plain HTML Sites:**

When using `azd` instead of raw `swa` commands for plain HTML sites, create this project structure:

```
my-static-site/
├── azure.yaml           # azd configuration
├── dist/                # Static content goes here (NOT in root)
│   └── index.html
└── infra/
    ├── main.bicep
    ├── main.parameters.json
    └── staticwebapp.bicep
```

**azure.yaml** - Note the critical settings:
```yaml
name: my-static-site
services:
  web:
    project: .
    # IMPORTANT: dist must point to a folder containing ONLY static content
    # Do NOT set dist to "." as it will include azure.yaml, infra/, .azure/, etc.
    # which causes deployment issues (stuck in "Uploading" state)
    dist: dist
    host: staticwebapp
    # IMPORTANT: Do NOT specify "language" for plain HTML sites
    # Setting language (e.g., "html", "js") causes errors like:
    # "language 'html' is not supported by built-in framework services"
```

**azure.yaml `language` property by host type:**
| Host Type | Language Property | Notes |
|-----------|-------------------|-------|
| `staticwebapp` | **Do NOT specify** for plain HTML | Only needed for managed Functions API |
| `containerapp` | Optional - used for build detection | Values: `js`, `ts`, `python`, `csharp`, `java`, `go` |
| `appservice` | Required for runtime selection | Values: `js`, `ts`, `python`, `csharp`, `java` |
| `function` | Required for runtime | Values: `js`, `ts`, `python`, `csharp`, `java` |

> 💡 **Validate azure.yaml before deployment** using the MCP tool:
> ```javascript
> const validation = await azure-azd({
>   command: "validate_azure_yaml",
>   parameters: { path: "./azure.yaml" }
> });
> ```

**infra/staticwebapp.bicep** - Must include the `azd-service-name` tag:
```bicep
@description('Name of the Static Web App')
param name string

@description('Location for the Static Web App')
param location string

resource staticWebApp 'Microsoft.Web/staticSites@2022-09-01' = {
  name: name
  location: location
  // REQUIRED: This tag links the Bicep resource to the service in azure.yaml
  tags: {
    'azd-service-name': 'web'
  }
  sku: {
    name: 'Free'
    tier: 'Free'
  }
  properties: {}
}

output name string = staticWebApp.name
output uri string = 'https://${staticWebApp.properties.defaultHostname}'
```

Then deploy with:
```bash
azd up
```

**Smart defaults:**
- SKU: `Free` for dev/test, `Standard` for production
- Location: SWA has limited regions - use `centralus`, `eastus2`, `westus2`, `westeurope`, or `eastasia`

See [Static Web Apps Guide](./reference/static-web-apps.md) for detailed configuration.

### 4.2 Azure Functions Deployment

> 💡 **For advanced Functions deployment scenarios**, see the **[Azure Functions Deployment Guide](./reference/functions.md)** which provides:
> - Function project initialization and templating
> - Detailed trigger/binding configuration
> - Deployment slots and CI/CD patterns
> - Function-specific troubleshooting
> - Complete MCP tool integration

**Create and deploy:**
```bash
# Create resource group
az group create --name <resource-group> --location <location>

# Create storage account (required for Functions)
az storage account create \
  --name <storage-name> \
  --resource-group <resource-group> \
  --location <location> \
  --sku Standard_LRS

# Create Function App
az functionapp create \
  --name <app-name> \
  --resource-group <resource-group> \
  --storage-account <storage-name> \
  --consumption-plan-location <location> \
  --runtime <node|python|dotnet|java> \
  --runtime-version <version> \
  --functions-version 4

# Deploy with func CLI
func azure functionapp publish <app-name>
```

**Smart defaults:**
- Plan: Consumption (pay-per-execution) for most cases
- Runtime version: Latest LTS for the detected language

See [Azure Functions Guide](./reference/functions.md) for advanced scenarios.

### 4.3 App Service Deployment

**Create and deploy:**
```bash
# Create resource group
az group create --name <resource-group> --location <location>

# Create App Service plan
az appservice plan create \
  --name <plan-name> \
  --resource-group <resource-group> \
  --location <location> \
  --sku B1 \
  --is-linux

# Create Web App
az webapp create \
  --name <app-name> \
  --resource-group <resource-group> \
  --plan <plan-name> \
  --runtime "<runtime>"

# Deploy code (zip deploy)
az webapp deploy \
  --name <app-name> \
  --resource-group <resource-group> \
  --src-path <path-to-zip> \
  --type zip
```

**Runtime values by language:**
- Node.js: `"NODE:18-lts"`, `"NODE:20-lts"`
- Python: `"PYTHON:3.11"`, `"PYTHON:3.12"`
- .NET: `"DOTNETCORE:8.0"`
- Java: `"JAVA:17-java17"`

**Smart defaults:**
- Plan SKU: `B1` for dev/test, `P1v3` for production
- Always use Linux (`--is-linux`) unless .NET Framework required

See [App Service Guide](./reference/app-service.md) for configuration options.

---

## Phase 5: Multi-Service Deployment (azd + IaC)

When multiple services or infrastructure dependencies are detected, recommend Azure Developer CLI with Infrastructure as Code.

### When to Use azd
- Multiple deployable components (frontend + API + functions)
- Needs database, cache, storage, or messaging services
- Requires consistent environments (dev, staging, production)
- Team collaboration with reproducible infrastructure

### Initialize azd Project
```bash
# Initialize from scratch
azd init

# Or use a template
azd init --template <template-name>
```

### Project Structure
```
project/
├── azure.yaml              # azd configuration
├── infra/
│   ├── main.bicep          # Main infrastructure
│   ├── main.parameters.json
│   └── modules/            # Reusable modules
├── src/
│   ├── web/                # Frontend
│   └── api/                # Backend
```

### Deploy with azd

**Step 1: Validate azure.yaml before deployment**

Always validate the azure.yaml configuration before running deployment commands:

```javascript
// Validate azure.yaml using MCP tool
const validation = await azure-azd({
  command: "validate_azure_yaml",
  parameters: { path: "./azure.yaml" }
});

// Check for validation errors
if (validation.errors && validation.errors.length > 0) {
  // Report errors to user and fix before proceeding
  console.error("azure.yaml validation failed:", validation.errors);
  return;
}
```

**Step 2: Deploy**

```bash
# Provision infrastructure + deploy code
azd up

# For automation/agent scenarios - use --no-prompt to avoid interactive prompts
azd up --no-prompt

# Or separately:
azd provision --no-prompt  # Create infrastructure
azd deploy --no-prompt     # Deploy application code

# Preview changes before deployment
azd provision --preview

# Manage environments
azd env new staging
azd env select staging
azd up --no-prompt
```

> ⚠️ **CRITICAL for automation**: Always use `--no-prompt` when azd is called by an agent or in CI/CD pipelines where interactive prompts cannot be answered.

**Step 3: On errors, use MCP troubleshooting**

```javascript
// If azd commands fail, get troubleshooting guidance
const troubleshooting = await azure-azd({
  command: "error_troubleshooting"
});
```

See [Multi-Service Guide](./reference/multi-service.md) for azure.yaml configuration.
See [Azure Verified Modules](./reference/azure-verified-modules.md) for Bicep module reference.

---

## Troubleshooting Quick Reference

### Common Issues

**"Not logged in" errors:**
```bash
az login
az account set --subscription "<name>"
```

**"Resource group not found":**
```bash
az group create --name <name> --location <location>
```

**SWA deployment fails:**
- Check build output directory is correct
- Verify deployment token is valid
- Ensure `staticwebapp.config.json` is properly formatted

**Functions deployment fails:**
- Verify `host.json` exists
- Check runtime version matches function app configuration
- Ensure storage account is accessible

**App Service deployment fails:**
- Verify runtime matches application
- Check startup command if using custom entry point
- Review deployment logs: `az webapp log tail --name <app> --resource-group <rg>`

See [Troubleshooting Guide](./reference/troubleshooting.md) for detailed solutions.

---

## Deployment Reference Guides

For specialized deployment scenarios, use these comprehensive reference guides:

- **🌐 [Static Web Apps Deployment Guide](./reference/static-web-apps.md)** - Static Web Apps deployment with SWA CLI, local emulation, GitHub Actions, API integration, and database connections
- **🐳 [Container Apps Deployment Guide](./reference/container-apps.md)** - Container Apps deployment with Docker validation, ACR integration, Container Apps Jobs, and multi-container orchestration
- **⚡ [Azure Functions Deployment Guide](./reference/functions.md)** - Azure Functions deployment with func CLI, triggers/bindings, deployment slots, and function-specific troubleshooting
- **☸️ [AKS Deployment Guide](./reference/aks.md)** - Kubernetes deployments with full control, custom operators, and complex microservices
- **🌍 [App Service Deployment Guide](./reference/app-service.md)** - Traditional web applications and REST APIs with managed hosting
- **🔌 [APIM Deployment Guide](../../azure-prepare/references/apim.md)** - API Management deployment for API Gateway and AI Gateway scenarios with Basicv2 SKU

---

Load these guides as needed for detailed information:

- [Static Web Apps Guide](./reference/static-web-apps.md) - Static frontends and JAMstack apps with SWA CLI, managed Functions APIs, GitHub Actions, and authentication
- [Container Apps Guide](./reference/container-apps.md) - Comprehensive Container Apps deployment with azd, Docker best practices, Jobs, scaling, and troubleshooting
- [Azure Functions Guide](./reference/functions.md) - Serverless Functions deployment with func CLI, triggers/bindings, deployment slots, and monitoring
- [AKS Guide](./reference/aks.md) - Kubernetes deployment with AKS, node pools, workload identity, scaling, and networking
- [App Service Guide](./reference/app-service.md) - Traditional web app deployment with App Service plans, deployment slots, and auto-scaling
- [APIM Guide](../../azure-prepare/references/apim.md) - API Management deployment for API gateway and AI gateway, with Basicv2 SKU for fast deployment
- Always scan the workspace before generating a deployment plan
- Plans integrate with Azure Developer CLI (azd)
- Logs require resources deployed through azd
- Run `/azure:preflight` before any deployment to avoid mid-deployment failures
