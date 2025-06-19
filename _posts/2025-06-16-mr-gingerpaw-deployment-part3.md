---
  title: Mr. Gingerpaw Project Log - Deployment Part 3 - Backend
  description: >-
    A development log for the Mr. Gingerpaw Inventory App. This entry focuses on backend deployment via Azure Function App and setting up automatic CI/CD workflows.
  # author: pointer
  date: 2025-06-16 12:38:00 +0930
  categories: [Blogging, MrGingerpaw]
  tags: [Python, Deployment, backend, FastAPI, Azure]
  # media_subpath: /images/2022-08-10-ipv6-version-of-frp/
  pin: false
---

## Introduction

The app has three major components: **Database**, **Backend**, and **Frontend**. All of them are deployed to Azure, with the goal of keeping costs as low as possible — ideally within the free tier.

For the backend, I chose **Azure Function App**. My backend is built with **FastAPI**, which integrates well with Azure Functions.

The following steps assume you already have an Azure account and can access the Azure Portal.

## Service Creation

Search for `Function App` under `Services`, click “Create,” and fill in the required fields. Here’s what each option means and what I chose.

### Basics

* **Subscription & Resource Group:** These organize your Azure resources. I reused the same resource group created for the Database.

* **Function App name:** Choose a descriptive name.

* **Region:** I picked **Southeast Asia**, as it's geographically close to China and Australia.

* **Runtime stack:** `Python`, since the backend uses FastAPI.

* **Version:** `3.10`. It’s generally recommended to stick with this for Azure Functions.

* **Instance size:** `2048 Mb`.

* **Zone redundancy:** `Disabled` (not supported in free tier).

### Storage

* **Storage account:** Automatically generated — no need to modify.
* **Diagnostic Settings:** Chose “Don’t configure now.” You can enable it later if needed.

### Azure OpenAI

Skipped, since I’m not using Azure's built-in OpenAI services. If your OpenAI usage isn’t via Azure, no setup is needed here.

### Networking

* **Enable public access:** `On` — useful for frontend to reach the backend during development.
* **Enable virtual network integration:** `Off` — no VNet setup required.

### Monitoring

* **Enable Application Insights:** I chose `Off` here, but **I do recommend enabling it**. I ended up turning it on later for debugging. This creates a new logging service and is very helpful.

### Deployment

* **Continuous deployment:** `On`.
* **GitHub settings:** Connect your GitHub account, and select the org/repo/branch.
* **Basic authentication:** `Off` — not required unless you’ve set up Basic Auth on GitHub.

> You can preview the generated `.yml` file here, but it's okay to adjust it later. Templates will need tweaking anyway.
{: .prompt-info }

### Authentication

Left as default.

### Tags

Added tags like `owner` and `project` for easier future management.

---

Once all fields are completed, go to **Review + Create**, check everything, and click **Create**.

## Modifying FastAPI for Azure Function

To deploy FastAPI on Azure Functions, a few changes are needed. See Microsoft’s sample guide here: [Using FastAPI Framework with Azure Functions](https://learn.microsoft.com/en-us/samples/azure-samples/fastapi-on-azure-functions/fastapi-on-azure-functions/).

### Step 1: Add Dependencies

In your `requirements.txt`, add:

```txt
azure-functions
```

### Step 2: Add the following files

* `backend/host.json`
* `backend/HttpTrigger/function.json`
* `backend/HttpTrigger/__init__.py`

```json
// backend/host.json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.0.0, 5.0.0)"
  },
  "extensions": {
    "http": {
      "routePrefix": ""
    }
  }
}
```

```json
// backend/HttpTrigger/function.json
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "route": "{*segment}",
      "methods": ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    }
  ],
  "scriptFile": "__init__.py"
}
```

```python
# backend/HttpTrigger/__init__.py

import azure.functions as func
from azure.functions import AsgiMiddleware
from app.main import app  # Your FastAPI app instance

asgi_app = AsgiMiddleware(app, http)

def main(req: func.HttpRequest, context: func.Context) -> func.HttpResponse:
    return asgi_app.handle(req, context)
```

Once this is done, your FastAPI backend is ready for deployment as a Function App. You should see `HttpTrigger` listed in your Function App dashboard.

## GitHub Workflow

If GitHub integration was set up correctly, a default workflow YAML will be created under `.github/workflows/`. Here’s the default template:

```yml
...
jobs:
  build:
    ...

  deploy:
    ...
```

However, my repo contains both frontend and backend under `~/frontend` and `~/backend`, so I modified the workflow to only deploy backend and skip the build step. Here's my version:

```yml
# .github/workflows/deploy-backend.yml

name: Build and Deploy backend to Azure Functions

on:
  push:
    branches: [ main ]
    paths:
      - 'backend/**'
      - '.github/workflows/deploy-backend.yml'
  workflow_dispatch:

env:
  AZURE_FUNCTIONAPP_PATH: './backend'

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZUREAPPSERVICE_CLIENTID_XXX }}
          tenant-id: ${{ secrets.AZUREAPPSERVICE_TENANTID_XXX }}
          subscription-id: ${{ secrets.AZUREAPPSERVICE_SUBSCRIPTIONID_XXX }}

      - name: Deploy to Azure Functions
        uses: Azure/functions-action@v1
        with:
          app-name: '[MyServiceName]'
          slot-name: 'Production'
          package: ${{ env.AZURE_FUNCTIONAPP_PATH }}
          sku: 'flexconsumption'
          remote-build: true
```

> **Important:** This workflow doesn’t handle local builds. You must let Azure handle the build process. Make sure you include both `sku: 'flexconsumption'` and `remote-build: true`. These are required for free-tier deployment.
{: .prompt-warning }

## CORS Settings

By default, Azure Function App doesn’t allow cross-origin requests. You must explicitly add your frontend domain(s) to the CORS list. For example:

```txt
# Dev
http://localhost:1234
http://192.168.1.128:1234

# Prod
https://mrginger.top
https://www.mrginger.top
```

Note: FastAPI’s internal CORS settings won’t take effect — Azure overrides them.

## Environment Variables

I manage environment variables directly in the Azure portal.

During local dev, my app reads variables from `.env`, but that file isn’t present in production. So I configure them in Azure’s environment settings instead.

> I initially tried setting env vars in GitHub and injecting them via workflow, but that didn’t work. Since the build runs in Azure, defining them there makes more sense.
{: .prompt-warning }

That’s it!
