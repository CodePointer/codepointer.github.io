---
  title: Mr. Gingerpaw Project Log - Deployment Part 4 - Frontend
  description: >-
    A development log for the Mr. Gingerpaw Inventory App. This entry focuses on frontend deployment using Azure Static Web Apps and setting up automated CI/CD workflows.
  # author: pointer
  date: 2025-06-19 23:33:00 +0930
  categories: [Blogging, MrGingerpaw]
  tags: [React, Deployment, Frontend, Azure]
  # media_subpath: /images/2022-08-10-ipv6-version-of-frp/
  pin: false
---


## Introduction

The app has three major components: **Database**, **Backend**, and **Frontend** — all deployed to Azure, with the goal of minimizing cost (ideally free).

For the frontend, I used **Azure Static Web Apps**, which offers generous free-tier support for hosting static sites.

The following assumes you’ve already signed up for Azure and can log in to the Azure Portal.

## Service Creation

Search for `Azure Static Web Apps` under **Services**, then click “Create” and follow the setup prompts. Here’s a breakdown of each field and what I selected.

### Basics

* **Subscription & Resource Group:** These manage billing and group related services. I placed the frontend in the same group as the backend and database.

* **Hosting region:** Automatically set to `Global` — no need to choose.

* **Static Web App name:** Pick any name you like.

* **Hosting plan:** Select `Free`.

* **Deployment details:** Choose GitHub, then select the organization, repository, and branch.

* **Build Presets:** Although I’m using React, I manually export the site. So this should be set to `Custom`.

* **App location:** This points to the folder containing your built static files. I used `./frontend/dist`.

* **API location & Output location:** Leave blank — build happens in GitHub.

### Deployment Configuration

* **Deployment authorization policy:** Use GitHub.

### Advanced & Tags

* **Region for Azure Functions API:** I picked `East Asia`, as the backend is hosted in Southeast Asia.
* **Enterprise-grade edge:** Not available on free tier.
* **Tags:** I added `owner` and `project` for organization.

---

Once everything is ready, review the config and click **Create**.

## GitHub Workflow

Once linked with GitHub, a YAML file is created under `.github/workflows/`. You’ll also find the filename listed on your Static Web App’s Overview page.

> **Important:** Do not rename this file. If it’s called `azure-static-web-apps-thankful-cliff-xxx.yml`, that name must remain unchanged. Otherwise, deployment will fail — this is a known issue since 2021.
{: .prompt-warning }

Since I need to build the code myself, I made the following adjustments to the generated YAML. Here's my custom workflow:

```yml
# .github/workflows/azure-static-web-apps-thankful-cliff-xxx.yml

name: Build and Deploy frontend to Azure Static Web Apps

on:
  push:
    branches: [ main ]
    paths:
      - 'frontend/**'
      - '.github/workflows/azure-static-web-apps-thankful-cliff-xxx.yml'
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    env:
      EXPO_PUBLIC_API_BASE_URL: ${{ secrets.EXPO_PUBLIC_API_BASE_URL }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          submodules: true
          lfs: false

      - name: Install OIDC Client from Core Package
        run: npm install @actions/core@1.6.0 @actions/http-client

      - name: Get Id Token
        uses: actions/github-script@v6
        id: idtoken
        with:
          script: |
              const coredemo = require('@actions/core')
              return await coredemo.getIDToken()
          result-encoding: string

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '22.x'

      - name: Install dependencies
        run: |
          cd frontend
          npm ci

      - name: Build Expo Web
        run: |
          cd frontend
          npx expo export --platform web

      - name: Deploy to Azure Static Web Apps
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_THANKFUL_CLIFF_xxx }}
          action: "upload"
          app_location: "./frontend/dist"
          api_location: ""
          output_location: ""
          github_id_token: ${{ steps.idtoken.outputs.result }}
          skip_app_build: true
```

## Custom Domain Binding

After purchasing a domain via NameSilo, I configured the DNS in Cloudflare. Azure recommends using a **CNAME** record. Once set up, return to the Static Web App service and complete **domain validation** to enable HTTPS.

> Without validation, HTTPS will not work properly and trigger certificate errors.
{: .prompt-tip }

> In my case, DNS propagation via Opticomm was extremely slow — it took over 24 to 48 hours to take effect.
{: .prompt-warning }

## Backend CORS Settings

Although this is technically a backend task, once the frontend is deployed, the backend must allow requests from the deployed frontend URLs. Don’t forget to update **CORS** settings on the backend to include the frontend’s domain.

## Final Thoughts

After completing all deployment steps, the app is now fully running online. Future updates can be made by simply pushing changes to the repo — deployments will happen automatically.

Eventually, I still plan to write some test units to safeguard basic functionality, but that’s a story for another day.


That’s it!
