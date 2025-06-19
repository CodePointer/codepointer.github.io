---
  title: Mr. Gingerpaw Project Log - Deployment Part 1 - Summary
  description: >-
    A development log for the Mr. Gingerpaw Inventory App. This deployment chapter focuses on planning and preparations for deploying the app to Azure with CI/CD workflows.
  # author: pointer
  date: 2025-06-12 14:15:00 +0930
  categories: [Blogging, MrGingerpaw]
  tags: [Python, React, Deployment, Azure]
  # media_subpath: /images/2022-08-10-ipv6-version-of-frp/
  pin: false
---

## Introduction

### Background

Since becoming a full-time home cook, I’ve found grocery shopping to be surprisingly tricky and tiring. It’s not uncommon to return from a trip only to realize we’re missing something important and need to head out again. In China that’s manageable, but here in Australia it’s much more of a hassle.

So, I decided to build an app to manage everything we have at home and track what needs restocking.

The app is now live in its very early form (`v0.0.1`). While it’s still basic, it covers our core needs and will be gradually improved over time. Today’s post is about how I deployed it to Azure.

As this is my first full-stack app, I plan to write more posts about its various technical details in the future.

### Requirements

The app consists of three main components: **Database**, **Backend**, and **Frontend**. All three need to be deployed to Azure.

Since this is a small personal project, I want to minimize costs — ideally keeping it entirely within Azure’s free tier.

Additionally, I want the deployment process to be automated. After making changes and testing locally, I should be able to deploy with a single click — no manual work.

## Deployment Plan

Here’s the solution I ultimately chose:

### Database: PostgreSQL Flexible Server

Since the app uses PostgreSQL, I opted for **Azure Database for PostgreSQL Flexible Server**.

This service is free for the first 12 months for new Azure accounts. It includes 750 compute hours and 32GB storage per month. After that, the estimated base cost is about USD \$23.40/month.

If that becomes too costly later, I might switch to another provider or move the database to my own VPS. But for now, this setup works.

### Backend: Function App

Specifically, I use **Azure Functions App**. My backend is built with Python FastAPI, which can be adapted easily to run on Function Apps.

An alternative would’ve been **Azure App Service**, but based on my use case, I chose Function App. The main differences are:

* **Azure Functions App** hosts functions that are triggered on demand and billed by invocation count.
* **Azure App Service** keeps the backend always online and bills based on runtime.

The App Service free tier only includes 60 CPU minutes per day, while Function App allows 100,000 invocations daily. For a few friends testing the app, 60 minutes isn’t quite enough.

> That said, Function App comes with a **cold start delay** — the first call after a period of inactivity takes 3–4 seconds to respond. In my testing, it’s noticeable.But by optimizing frontend logic and using local drafts to reduce API calls, I think the delay is manageable.
{: .prompt-tip}

### Frontend: Static Web App

Azure **Static Web Apps** is perfect for hosting static React apps. The free tier is generous, and I don’t expect to incur any cost.

My frontend is built with React + Expo and can be bundled as a static site to be deployed here.

I also plan to purchase a custom domain for the frontend, likely from **NameSilo** (cheapest option), and manage DNS with **Cloudflare**, which I’m used to.

### Deployment: GitHub Workflows

Since my code is hosted on GitHub, GitHub Actions is the most natural choice for automation.

---

Given the number of issues and decisions involved in deployment, I’ll be splitting this topic into three separate blog posts:

- Database: [Deployment Part 2: Database](/posts/mr-gingerpaw-deployment-part2)
- Frontend: [Deployment Part 3: Backend](/posts/mr-gingerpaw-deployment-part3)
- Backend: [Deployment Part 4: Frontend](/posts/mr-gingerpaw-deployment-part4)


That’s it!
