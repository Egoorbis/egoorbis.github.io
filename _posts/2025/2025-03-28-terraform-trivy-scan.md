---
title: Fortify Your Terraform - Scanning IaC with Trivy and GitLeaks on GitHub Actions
date: 2025-03-28 20:30:SS +0200
categories: [Infrastructure as Code, Terraform]
tags: [terraform, vulnerability-management, GitHub actions, trivy, giteaks]
description: Secure your Terraform deployments by catching vulnerabilities early. Learn how to integrate Trivy and GitLeaks into your GitHub Actions workflow, enhancing your IaC security posture.
---

## Introduction: Securing Your Infrastructure from the Start

In today's fast-paced development landscape, security cannot be an afterthought. Ensuring the integrity of your Infrastructure as Code (IaC) is paramount, especially when working with tools like Terraform. Enter Trivy<sup>[1](#sources)</sup>, a powerful open-source security scanner designed to detect vulnerabilities across various targets, and GitLeaks<sup>[2](#sources)</sup>, an open-source scanner optimized to identify secrets.

n this post, you'll learn how to use Trivy in GitHub Actions to scan your Terraform code for vulnerabilities before deploying. While working on this, I found Trivy's secret detection had some gaps, so I added GitLeaks to the mix

## Prerequisites: Setting the Stage

For this tutorial, you'll need a GitHub repository. I'll build upon the setup from my previous blog post, which detailed OIDC and Entra ID authentication for Azure deployments from GitHub Actions. You can find the details [here](https://egoorbis.github.io/2024/2025-01-31-terraform-azure-deployment).

TThe files used in this post are available in my blog-data repository:
* [Terraform Code](https://github.com/Egoorbis/blog-data/tree/main/trivy-scan)
* [GitHub Actions Workflows](https://github.com/Egoorbis/blog-data/blob/main/.github/workflows)

## GitHub Actions Workflow: Building the Foundation

First, create a new GitHub Actions workflow. If needed, create the `.github/workflows` directory and add the following YAML file.

Enhanced from my earlier blog post, this workflow integrates a basic Trivy scan, which I'll explain in the following section.

```yaml
name: "terraform-trivy-basic-scan"

on:
  push:
    branches:
      - main

permissions:
  contents: read
  id-token: write

jobs:
  terraform:
    name: "Terraform"
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash
        working-directory: ${{ env.TERRAFORM_DIR }}
    env:
      TERRAFORM_DIR: trivy-scan 
      TERRAFORM_LOG: "WARN"
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      BACKEND_RESOURCE_GROUP: ${{ secrets.BACKEND_RESOURCE_GROUP }}
      BACKEND_STORAGE_ACCOUNT: ${{ secrets.BACKEND_STORAGE_ACCOUNT }}
      BACKEND_CONTAINER_NAME: ${{ secrets.BACKEND_CONTAINER_NAME }}
      BACKEND_KEY: ${{ secrets.BACKEND_KEY }}

    steps:
      - name: "Code Checkout"
        uses: actions/checkout@v4

      - name: Run Trivy IaC scan
        uses: aquasecurity/trivy-action@master 
        with:
          scan-type: 'fs'
          scan-ref: ${{ env.TERRAFORM_DIR }}
          scanners: secret,misconfig
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
          skip-dirs: '.terraform'
          trivyignores: ${{ env.TERRAFORM_DIR }}/.trivyignore
          
      - name: "Install Terraform"
        uses: hashicorp/setup-terraform@v3

      - name: "Terraform Version"
        id: version
        run: terraform --version

      - name: "Terraform Init"
        id: init
        run: |
          terraform init \
            -backend-config="resource_group_name=$BACKEND_RESOURCE_GROUP" \
            -backend-config="storage_account_name=$BACKEND_STORAGE_ACCOUNT" \
            -backend-config="container_name=$BACKEND_CONTAINER_NAME" \
            -backend-config="key=$BACKEND_KEY" \
      - name: "Terraform Plan"
        id: plan
        run: |
          terraform plan -out=tfplan
      - name: "Upload Terraform Plan to Working Directory"
        uses: actions/upload-artifact@v4
        with:
          name: terraformPlan
          path: "tfplan"

      - name: "Terraform Apply using Plan File"
        id: apply
        run: terraform apply tfplan
```

## Integrate Trivy Scan into the Workflow

We'll integrate Trivy using the [aquasecurity/trivy-action](https://github.com/aquasecurity/trivy-action). This step is added immediately after the "Code Checkout" to scan the Terraform code before execution.

Let's explore Trivy's versatile options for scanning IaC configurations. I highly recommend experimenting locally to tailor the settings to your specific needs. Trivy offers flexibility, allowing you to define options directly in your pipeline or via a YAML file within your repository. For comprehensive details, consult the Trivy IaC documentation<sup>[2](#sources)</sup> and the Trivy action documentation<sup>[3](#sources)</sup> 

* `scan-type`: Specifies the scan type (fs for filesystem scans).
* `scan-ref`: Sets the desired Terraform directory - without this Trivy will scan the whole repository.
* `scanners`: Defines the types of security issues to detect (IaC supports misconfiguration and secrets).
* `format`: Specifies the output format.
* `exit-code`: Fails the workflow if vulnerabilities are found.
* `severity`: Sets the severity level to check.
* `skip-dirs`: Excludes specified directories.
* `trivyignores`: Specifies a `.trivyignore` file to ignore specific issues by adding the ID (See section [Ignoring Specific Issues](#ignoring-specific-issues)).

Let's review the step's output to understand the results, which are displayed as follows:

![Run Trivy Infrastructure as Code scan step](/assets/img/posts/2025-03-28-terraform-trivy-scan/trivy-pipeline.png)

While Trivy detected issues within our code, the output lacks readability. 

To improve this, we'll modify the configuration to publish the scan results to the GitHub Actions summary page, providing a better overview of all security vulnerabilities. 

Additionally, we'll configure a second Trivy scan to fail only if any High or Critical issues are detected. 

```yaml
      - name: Run Initial Trivy IaC scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: ${{ env.TERRAFORM_DIR }}
          scanners: secret,misconfig
          format: 'table'
          skip-dirs: '.terraform'
          trivyignores: ${{ env.TERRAFORM_DIR }}/.trivyignore
          hide-progress: true
          output: $GITHUB_WORKSPACE/trivy.txt

      - name: Publish Trivy Output to Summary
        run: |
          if [[ -s $GITHUB_WORKSPACE/trivy.txt ]]; then
            {
              echo "### Security Output"
              echo "<details><summary>Click to expand</summary>"
              echo ""
              echo '```terraform'
              cat $GITHUB_WORKSPACE/trivy.txt
              echo '```'
              echo "</details>"
            } >> $GITHUB_STEP_SUMMARY
          fi        

      - name: Run High, Critical Trivy IaC scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: ${{ env.TERRAFORM_DIR }}
          scanners: secret,misconfig
          hide-progress: true
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          skip-dirs: '.terraform'
          trivyignores: ${{ env.TERRAFORM_DIR }}/.trivyignore
          skip-setup-trivy: true
```

The output of this configuration is presented below:

![Trivy GitHub Summary](/assets/img/posts/2025-03-28-terraform-trivy-scan/trivy-github-summary.png)

## Ignoring Specific Issues

In the code above we've already incorporated `trivyignores: ${{ env.TERRAFORM_DIR }}/.trivyignore` into our Trivy action, enabling us to selectively exclude identified security issues. This is particularly useful for managing known or acceptable risks in your IaC.

For example, as I lack a fixed public IP address, I've chosen to exclude the "AVD-AZU-0041 (CRITICAL): Cluster does not limit API access to specific IP addresses" vulnerability. To achieve this, I simply add the ID `AVD-AZU-0041` to the `.trivyignore` file, placing each ignored vulnerability on a new line. Subsequent Trivy scans will then bypass these specified issues.

## Enhancing Secret Detection with GitLeaks

During initial Trivy scans, I observed that it failed to detect an intentionally embedded password within the [main.tf](https://github.com/Egoorbis/blog-data/blob/main/trivy-scan/main.tf) file.

This aligns with findings reported in a blog post by **Rooneycloudtech**<sup>[4](#sources)</sup>, which highlighted the same limitation.

Following the authors recommendation I've integrated GitLeaks my workflow to enhance secret detection, complementing Trivy. Using the official [GitLeaks action](https://github.com/gitleaks/gitleaks-action), the following configuration was used:

```yaml
 - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Surprisingly, the official GitLeaks GitHub Action failed to detect the secrets, reporting no issues, despite local testing successfully detecting the password.

![GitLeaks Action](/assets/img/posts/2025-03-28-terraform-trivy-scan/trivy-gitleaks-actions.png)

 To resolve this discrepancy, I've modified the workflow to manually install GitLeaks, as shown in the following configuration.

```yaml
      - name: Install Gitleaks
        run: |
          wget https://github.com/zricethezav/gitleaks/releases/download/v8.18.1/gitleaks_8.18.1_linux_x64.tar.gz
          tar -xzf gitleaks_8.18.1_linux_x64.tar.gz
          sudo mv gitleaks /usr/local/bin/
      - name: Run Gitleaks
        id: gitleaks
        run: |
          # Run gitleaks and strip ANSI escape codes
          gitleaks detect --source . -v --no-banner --no-color > gitleaks_output.txt
        continue-on-error: true
        
      - name: Add Gitleaks results to summary
        if: always()
        run: |
          echo "## Gitleaks Scan Results" >> $GITHUB_STEP_SUMMARY
          if [[ "${{ steps.gitleaks.outcome }}" == "failure" ]]; then
            echo "⚠️ Gitleaks found potential secrets in your code." >> $GITHUB_STEP_SUMMARY
            echo "### Detected Leaks:" >> $GITHUB_STEP_SUMMARY
            echo '```' >> $GITHUB_STEP_SUMMARY
            # Format and clean the output
            grep -v "^[[:space:]]*$" gitleaks_output.txt | \
            grep -v "^    ○" | \
            grep -v "^    │" | \
            grep -v "^    ░" | \
            grep -v "gitleaks$" >> $GITHUB_STEP_SUMMARY
            echo '```' >> $GITHUB_STEP_SUMMARY
          else
            echo "✅ No leaks detected by Gitleaks." >> $GITHUB_STEP_SUMMARY
          fi
```

As shown in the screenshot below, this time the password was successfully detected.

![GitLeaks Manual](/assets/img/posts/2025-03-28-terraform-trivy-scan/trivy-gitleaks-actions-manual.png)

Through the implementation of Trivy and GitLeaks in a GitHub Actions workflow, we have now established robust security our Terraform deployment, encompassing both misconfiguration and secret detection.

## Sources

1. [Trivy GitHub Repository](https://github.com/aquasecurity/trivy)

2. [GitLeaks](https://gitleaks.io/)

2. [Trivy Infrastructure as Code¶](https://trivy.dev/latest/docs/coverage/iac/)

3. [Trivy Action](https://github.com/aquasecurity/trivy-action)

4. [Rooneycloudtech - Trivy vs Gitleaks: Optimizing Repository Security in Modern DevOps.](https://www.rooneycloudtech.com/trivy-vs-gitleaks-optimizing-repository-security-in-modern-devops/)






