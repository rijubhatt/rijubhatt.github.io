# CI/CD Pipeline for Azure Data Factory (ADF)

## Purpose
This page provides detailed guidance on implementing a CI/CD pipeline for Azure Data Factory (ADF) in an event-driven ETL architecture. It also addresses missing aspects from existing documentation, including environment-specific credential injection during ARM deployment and consolidated permissions for Azure DevOps service connections using self-hosted agents.

---

## ADF Architecture in Event-Driven ETL Scenario
![ADF-ARCH](https://github.com/user-attachments/assets/762a887a-3402-4f14-9adb-859ecdd640bf)

**Scenario Overview:**
- **Azure Storage Blob:** Located in a public subnet, it serves as the source of data.
- **ADF Instance:** Resides in a private subnet and performs data ingestion and transformation.
- **Connection Establishment:** The ADF instance connects to the Azure Storage Blob using private endpoints or service endpoints to ensure secure communication.

---

## CI/CD Pipeline Overview

### Pipeline Flow
The pipeline is implemented in Azure DevOps and follows a structured flow for development, integration, and deployment:

#### Source Control
- **Repository:** Azure Repos is used as the source control system for ADF code.
- **Branching Strategy:** Developers create feature/* branches from the main branch for each new feature or update.

#### Development
- **ADF Environment:** Developers create ETL pipelines, datasets, and linked services directly within the ADF DEV environment (Git mode).
- **Commit Workflow:** Changes are committed to the corresponding feature/* branch in Azure Repos.

#### Pull Request (PR) and Merge
- Developers raise a PR to merge changes from feature/* into the main branch.
- Code reviews ensure quality and compliance.
- Upon approval, the feature/* branch is merged into the main branch.

![ADF-CICD](https://github.com/user-attachments/assets/478494eb-e66a-4e2a-a0ff-2641b43e0c2c)


### Continuous Integration (CI)

The Azure DevOps CI pipeline performs:
- Validation of ADF JSON templates.
- Generation of ARM templates for ETL pipelines, datasets, and linked services.
- Creation of a pipeline artifact containing the generated ARM templates.

### Continuous Deployment (CD)

Artifact creation triggers the CD pipeline, which:
1. **Stop Triggers:** Temporarily stops active triggers in the target environment.
2. **Update Parameters:** Customizes parameters and updates environment-specific configurations. This includes executing custom scripts to modify credentials (e.g., connection strings, API keys, service principal objects).
3. **Deploy ARM Template:** Deploys the ARM templates to the target ADF instance.
4. **Start Triggers:** Reactivates triggers after deployment.

### Deployment Flow

- **DEV:** Auto-deployment of ARM templates (Live mode).
- **TEST, PRE-PROD, PROD:** Promoted deployment upon approval for each subsequent environment (Live mode).

---

## Technical Requirements

### Tools and Services
- **Azure DevOps:** For CI/CD pipeline orchestration.
- **Azure Repos:** For version control.
- **Self-Hosted Agents:** Used in Azure DevOps pipelines for interaction with Azure resources.

### Environment Configurations
- **Storage Account:** Located in a public subnet with required permissions for secure data access.
- **ADF:** Configured in a private subnet.

### Access and Permissions
The following consolidated permissions are required for the service connection:
- **Azure Data Factory:** Contributor role.
- **Azure Storage Account:** Contributor role.
- **Key Vault:** Secrets Officer role (for managing sensitive data like credentials).
- **Resource Group:** Contributor role (for deployment activities).

---

## Repository Structure

### Folder Layout
```plaintext
data-factory:
  credential/          # Environment-specific credentials (e.g., connection strings, API keys).
  dataflow/            # Data flow definitions for transformations.
  dataset/             # Dataset definitions for source/target structures.
  linkedService/       # Linked service definitions for external connections.
  pipeline/            # Pipeline definitions for orchestration.
  trigger/             # Trigger definitions for event-based executions.
devops:
  package/             # Details for ADFUtilities package.
  pipelines/           # YAML pipeline definitions.
  templates/
    build/             # Build template for common tasks.
    deploy/            # Deployment template for common tasks.
```

---

## Key Emphasis

1. **Environment-Specific Credentials Injection:**
   - Add a custom script to modify `credentials` JSON files during the deployment stage to handle environment-specific connection strings, API keys, or service principal objects.
   Example of such a script:
```plaintext
- task: PythonScript@0
  displayName: 'Refresh resource ID of ManagedIdentity'
  env:
    NEW_RESOURCE_ID: '/subscriptions/${{ parameters.subId }}/resourcegroups/${{ parameters.rgName }}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${{ parameters.userMgdIdtyName }}'
  inputs:
    scriptSource: 'inline'
    script: |
      import json
      import os

      # Path to the ARM template
      template_path = "ARMTemplateForFactory.json"

      # Retrieve the new resource ID from the environment variable
      new_resource_id = os.getenv('NEW_RESOURCE_ID')

      # Load the JSON content
      with open(template_path, "r") as file:
          template_content = json.load(file)

      # Update resourceId only after the specific type
      for resource in template_content.get("resources", []):
          if resource.get("type") == "Microsoft.DataFactory/factories/credentials":
              if resource.get("properties", {}).get("typeProperties", {}).get("resourceId"):
                  resource["properties"]["typeProperties"]["resourceId"] = new_resource_id

      # Save the updated JSON back to the file
      with open(template_path, "w") as file:
          json.dump(template_content, file, indent=2)

      print(f"Updated resourceId to {new_resource_id} in {template_path}")
```

2. **Access and Permissions:**
   - Ensure consolidated and precise permission allocation to service connections for seamless CI/CD execution.

---

## References
- [Azure Data Factory CI/CD Improvements](https://learn.microsoft.com/en-us/azure/data-factory/continuous-integration-delivery-improvements#the-new-cicd-flow)
- [Storage Event Trigger - Permission and RBAC Settings](https://techcommunity.microsoft.com/blog/azuredatafactoryblog/storage-event-trigger---permission-and-rbac-setting/2101782)
