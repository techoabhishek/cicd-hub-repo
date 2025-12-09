# CI/CD Hub Repository

This repository acts as the central "Hub" in a Hub-and-Spoke CI/CD model using GitHub Actions. It contains reusable and standardized workflows for building, testing, and deploying Node.js applications to Google Cloud Run.

## Hub-and-Spoke Model Overview

This model centralizes CI/CD logic to ensure consistency, security, and maintainability across multiple application repositories ("Spokes").

*   **Hub Repository (This Repo):** Contains generic, reusable `workflow_call` workflows that define the core steps of the CI/CD process (e.g., build, test, deploy).
*   **Spoke Repositories:** These are your individual application repositories. They contain their own GitHub Actions workflows that simply *call* the reusable workflows from this Hub, passing in application-specific parameters.

This approach has several benefits:
*   **Consistency:** All applications are built and deployed using the same, tested process.
*   **Maintainability:** CI/CD logic is updated in one place (the Hub), and all Spoke projects automatically inherit the changes.
*   **Security:** Centralizes management of sensitive steps like authentication and deployment.
*   **Simplicity:** Spoke repository workflows are minimal and declarative, focusing only on what is unique to their application.

---

## Available Reusable Workflows

This Hub provides the following workflows located in `.github/workflows/`:

### 1. `nodejs-ci.yaml` (Pull Request Validation)

This workflow is designed to be triggered on `pull_request` events in a Spoke repository. It validates the code changes before they are merged.

*   **Purpose:** To run a series of checks on a pull request to ensure code quality and that the application is in a buildable state.
*   **Key Steps:**
    1.  Checks out the code.
    2.  Installs dependencies using `npm ci` for a clean, reproducible install.
    3.  Runs tests using `npm test`.
    4.  Performs a `docker build` as a dry run to validate the `Dockerfile` without pushing the image.
    5.  Sends a Slack notification upon successful completion.

### 2. `nodejs-cloudrun-deploy.yaml` (Build & Deploy to Cloud Run)

This workflow is designed to be triggered on `push` events (e.g., to `main` or `develop` branches) in a Spoke repository.

*   **Purpose:** To build a container image, push it to Google Artifact Registry, and deploy it to Google Cloud Run.
*   **Key Steps:**
    1.  Checks out the code from the Spoke repository.
    2.  Authenticates to Google Cloud using Workload Identity Federation.
    3.  Installs dependencies and runs tests.
    4.  Builds the Docker image and pushes it to Google Artifact Registry.
    5.  Deploys the new image version to the specified Cloud Run service.
    6.  Sends a Slack notification with the final deployment status (success or failure).

---

## How to Integrate a Spoke Repository

Follow these steps to set up a new application repository ("Spoke") to use the workflows from this Hub.

### Prerequisites

1.  **Google Cloud Project:** You must have a GCP project with the following APIs enabled:
    *   Artifact Registry API (`artifactregistry.googleapis.com`)
    *   Cloud Run Admin API (`run.googleapis.com`)
    *   IAM API (`iam.googleapis.com`)

2.  **Artifact Registry:** Create a Docker repository in Google Artifact Registry to store your container images.

3.  **Workload Identity Federation:** Configure Workload Identity Federation between your GCP project and the Spoke GitHub repository. This allows GitHub Actions to securely authenticate to GCP without long-lived service account keys.
    *   Create a Workload Identity Pool and Provider.
    *   Create a Google Service Account (like `github-action-cicd@...`) with the following roles:
        *   `roles/artifactregistry.writer` (to push images)
        *   `roles/run.developer` (to deploy to Cloud Run)
        *   `roles/iam.serviceAccountUser` (to impersonate the service account)
    *   Grant the GitHub subject permission to impersonate this service account.

4.  **GitHub Secrets:** In your Spoke repository, go to `Settings > Secrets and variables > Actions` and add the following repository secret:
    *   `SLACK_WEBHOOK`: The incoming webhook URL for your Slack workspace to receive notifications.

### Step 1: Create the Spoke Workflow

In your Spoke repository, create a workflow file (e.g., `.github/workflows/main.yaml`). This file will define when the CI/CD process runs and will call the Hub workflows.

Below is a complete example that runs validation on pull requests and deploys on pushes to `main` (production) and `develop` (staging).

```yaml
# .github/workflows/main.yaml in your application (Spoke) repo

name: Spoke CI/CD - Deploy to Cloud Run

on:
  pull_request:
    branches:
      - main
      - develop
  push:
    branches:
      - main
      - develop
  workflow_dispatch:

jobs:
  # Job 1: Run validation checks for Pull Requests
  pr-check:
    if: github.event_name == 'pull_request'
    uses: techoabhishek/cicd-hub-repo/.github/workflows/nodejs-ci.yaml@main
    with:
      service_dir: . # Path to your app code within the repo
    secrets: inherit

  # Job 2: Determine environment for push events
  setup:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    outputs:
      env_name: ${{ steps.env.outputs.env_name }}
      notify_status: ${{ steps.env.outputs.notify_status }}
    steps:
      - id: env
        run: |
          if [[ ${{ github.ref }} == 'refs/heads/main' ]]; then
            echo "env_name=production" >> $GITHUB_OUTPUT
            echo "notify_status=approval" >> $GITHUB_OUTPUT
          elif [[ ${{ github.ref }} == 'refs/heads/develop' ]]; then
            echo "env_name=staging" >> $GITHUB_OUTPUT
            echo "notify_status=skip" >> $GITHUB_OUTPUT
          fi

  # Job 3: Call the reusable deployment workflow
  call-hub-deploy:
    needs: [setup]
    if: github.event_name == 'push' && needs.setup.result == 'success'
    permissions:
      contents: read
      id-token: write # Required for authenticating to Google Cloud
    uses: techoabhishek/cicd-hub-repo/.github/workflows/nodejs-cloudrun-deploy.yaml@main
    with:
      environment: ${{ needs.setup.outputs.env_name }}
      service_dir: .
      project_id: your-gcp-project-id
      region: us-central1
      image_repo: your-artifact-registry-repo
      service_name: your-cloud-run-service-name
    secrets: inherit # This passes the SLACK_WEBHOOK secret to the called workflow
```

### Step 2: Customize the `with` Block

In the `call-hub-deploy` job of your Spoke workflow, you must provide values for the `with` block. These are the parameters your specific application needs for deployment.

*   `environment`: The name of the GitHub Environment to use (e.g., `production`).
*   `service_dir`: The directory within your repository where the application source code and `Dockerfile` are located. Use `.` for the root.
*   `project_id`: Your Google Cloud Project ID.
*   `region`: The GCP region for your Artifact Registry and Cloud Run service (e.g., `us-central1`).
*   `image_repo`: The name of your repository in Google Artifact Registry.
*   `service_name`: The name of your service in Google Cloud Run.

And that's it! When you push a commit or create a pull request in your Spoke repository, it will now use the centralized logic from this Hub to run its CI/CD pipeline.