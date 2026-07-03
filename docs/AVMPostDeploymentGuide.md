---
title: Post-deployment guide
description: Steps to verify, secure, and populate Chat with Your Data after azd up provisions the Container Apps environment, Azure AI Foundry, and the ingestion worker.
ms.date: 2026-07-03
ms.topic: how-to
---

[Back to *Chat with your data* README](../README.md)

![Supporting documentation](images/supportingDocuments.png)

## Overview

This guide covers what to do after you deploy Chat with Your Data. The infrastructure is built from Azure Verified Modules and provisioned by `azd up`, which creates the Container Apps environment, the three application containers (web app, backend API, and ingestion worker), Azure AI Foundry, and the storage and database resources. For a full walkthrough of the deployment itself, see [Deploy with azd](LOCAL_DEPLOYMENT.md).

By the time `azd up` finishes, two hooks have already run:

* **Post-provision** prepares the data platform — for example, enabling the `pgvector` extension when you deploy in PostgreSQL mode.
* **Post-deploy** optionally seeds a sample scenario so chat works out of the box.

The steps below confirm the deployment, secure it, and add your own content.

## Prerequisites

* A successful `azd up` deployment. Run `azd env get-values` to see the outputs for your environment.
* Permission to configure authentication on the web app and to assign the admin role.

## Step 1: Confirm the deployment

Check the outputs for your environment and open the web app.

```bash
azd env get-values
```

Open the web app URL from the output in a browser. The chat page should load. If you seeded sample data during deployment, ask a question about it to confirm retrieval works.

## Step 2: Configure authentication

Restrict access so only authorized users can open the app.

1. Enable the built-in authentication on the web app's container app.
2. Follow [App authentication setup](azure_app_service_auth_setup.md) to configure the identity provider and assign the `admin` role to the people who manage content.

Administration lives inside the web app; there is no separate admin site. Users with the `admin` role see an admin area at `/admin`. See [Admin and configuration](admin.md).

## Step 3: Ingest your documents

1. Sign in as a user with the `admin` role and open the admin area.
2. On the **Ingest** page, upload documents or submit a URL. Sample documents are available under the `data/` directory of this repository.
3. The ingestion worker parses, chunks, embeds, and indexes the content. For how the pipeline works, see [Document ingestion](document_ingestion.md); for accepted formats, see [Supported file types](supported_file_types.md).

## Step 4: Test the chat experience

Open the chat page and ask a question about your uploaded content. The assistant answers with inline citations back to the source documents and streams a progress panel as it works. See [Streaming answers and citations](streaming_responses.md).

## Next steps

* **[Document ingestion](document_ingestion.md)** — understand how documents are processed and indexed.
* **[Admin and configuration](admin.md)** — ingest and manage documents from the built-in admin pages.
* **[Speech-to-text](speech_to_text.md)** — add voice interaction.
* **[Troubleshooting](TroubleShootingSteps.md)** — resolve common deployment errors.
