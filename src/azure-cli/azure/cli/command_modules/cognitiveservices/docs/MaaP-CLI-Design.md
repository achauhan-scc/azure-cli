# MaaP on Foundry – CLI Design Document

**Author:** Amit Chauhan  
**Date:** 2026-03  
**Reference:** MaaP on Foundry - User experiences (SeokJin Han, 2026-02)

---

## 1. Overview

This document describes the **CLI-only changes** for supporting Model as a Platform (MaaP) on Azure AI Foundry.

### Key Assumption

The CLI does **not** interact with the REST API directly. It uses the SDK (`azure-mgmt-cognitiveservices`) as the sole interface. The CLI implementation is:

1. **Interface change** — add new params and argument groups to the command
2. **Validation** — enforce rules (mutual exclusivity, conditional required)
3. **SDK wiring** — map CLI params to SDK model objects

All API contract details, serialization, and HTTP handling are the SDK's responsibility.

### Prerequisites

| Prerequisite | Description | Blocking? |
|-------------|-------------|-----------|
| SDK update (`azure-mgmt-cognitiveservices`) | Must expose new fields on `DeploymentModel` / `DeploymentProperties` for `deployment_template` and `accelerator_type` | **Yes** — CLI cannot ship without this |
| `DeploymentModel.source` support | SDK must accept `azureml://` URI as model source for `--model-id` | **Yes** |
| `Sku.name` value `GlobalManagedCompute` | SDK must accept this as a valid SKU name | **Yes** |

### Scope

| Area | CLI Module | Repo | What Changes |
|------|-----------|------|-------------|
| Consumer: Deploy models | `az cognitiveservices` | azure-cli (core) | `_params.py`, `custom.py`, `_help.py` |
| Publisher: Package models | `az ml` | azure-cli-extensions | YAML schema for DT and Model |

### What This Document Does NOT Cover

- API contract design (see: Foundry SKU support.docx)
- SDK implementation details
- Backend orchestration (MED, MRS, Singularity)
- UI changes

---

## 2. Consumer – `az cognitiveservices account deployment create`

### 2.1 Usage

**Current (ADM models):**
```bash
az cognitiveservices account deployment create \
  -g rg1 --name account1 --deployment-name dep1 \
  --model-format OpenAI --model-name gpt-4 --model-version 0613 \
  --sku-name Standard --sku-capacity 1
```

**New (MaaP / GlobalManagedCompute):**
```bash
az cognitiveservices account deployment create \
  -g rg1 --name account1 --deployment-name dep1 \
  --model-id azureml://registries/reg1/models/model1/versions/1 \
  --sku-name GlobalManagedCompute --sku-capacity 4 \
  --deployment-template azureml://registries/reg1/deploymenttemplates/dt1/versions/1 \
  --accelerator-type A100_80GB
```

### 2.2 Parameter Changes

#### New parameters

| Parameter | Argument Group | Required | Description |
|-----------|---------------|----------|-------------|
| `--model-id` | MaaP Deployment | When sku-name=GlobalManagedCompute | Registry model reference (`azureml://registries/{reg}/models/{model}/versions/{ver}`) |
| `--deployment-template` | MaaP Deployment | No | Deployment template reference (`azureml://registries/{reg}/deploymenttemplates/{dt}/versions/{ver}` or `.../labels/latest`) |
| `--accelerator-type` | MaaP Deployment | No | Accelerator type (e.g., `A100_80GB`, `H100_80GB`) |

#### Modified parameters

| Parameter | Change | Reason |
|-----------|--------|--------|
| `--model-format` | Required → Optional | Redundant for MaaP; model-id implies the publisher. Not a breaking change for existing ADM scenarios. |
| `--sku-name` | New allowed value: `GlobalManagedCompute` | Identifies MaaP deployment type |
| `--sku-capacity` | Semantic change when sku=GlobalManagedCompute | Represents "model instances" (shown as "model instance" in UI) |

### 2.3 Validation Rules

Validation is implemented in `custom.py` (consistent with Azure CLI patterns in AKS, ContainerApp, etc.).

| Rule | Error Message |
|------|--------------|
| `--sku-name GlobalManagedCompute` requires `--model-id` | `--model-id is required when --sku-name is GlobalManagedCompute.` |
| `--model-id` and `--model-name` are mutually exclusive | `--model-id and --model-name/--model-version are mutually exclusive. Use --model-id for GlobalManagedCompute deployments.` |
| `--deployment-template` / `--accelerator-type` only valid with GlobalManagedCompute | `--deployment-template and --accelerator-type are only applicable when --sku-name is GlobalManagedCompute.` |
| ADM deployments still require `--model-format`, `--model-name`, `--model-version` | `--model-format, --model-name, and --model-version are required for non-GlobalManagedCompute deployments.` |

### 2.4 CLI → SDK Mapping

CLI builds SDK model objects and calls `client.begin_create_or_update()`. The mapping:

| CLI Parameter | SDK Field | Notes |
|--------------|-----------|-------|
| `--model-id` | `DeploymentModel.source` | azureml:// registry reference |
| `--model-format` | `DeploymentModel.format` | Optional for MaaP |
| `--model-name` | `DeploymentModel.name` | ADM only |
| `--model-version` | `DeploymentModel.version` | ADM only |
| `--sku-name` | `Sku.name` | New value: `GlobalManagedCompute` |
| `--sku-capacity` | `Sku.capacity` | = model instances for GlobalManagedCompute |
| `--deployment-template` | TBD (pending SDK field) | |
| `--accelerator-type` | TBD (pending SDK field) | |

### 2.5 Help Output

Argument groups provide visual separation in `--help` so MaaP params are clearly distinct from ADM params.

```
Arguments
    --name -n               [Required] : Cognitive service account name.
    --resource-group -g     [Required] : Name of resource group.
    --deployment-name                  : Cognitive Services account deployment name.
    --capacity --sku-capacity          : Capacity value of the Sku (model instances
                                         for GlobalManagedCompute).
    --sku --sku-name                   : Name of the Sku. Allowed values include:
                                         Standard, GlobalStandard, GlobalManagedCompute, ...

DeploymentModel Arguments
    --model-format                     : Deployment model format. Required for
                                         non-GlobalManagedCompute SKUs.
    --model-name                       : Deployment model name.
    --model-version                    : Deployment model version.
    --model-source                     : Deployment model source.

MaaP Deployment Arguments
    --model-id                         : Registry model ID (azureml://...).
                                         Required when --sku-name is GlobalManagedCompute.
                                         Mutually exclusive with --model-name/--model-version.
    --deployment-template              : Deployment template reference (azureml://...).
                                         Only applicable when --sku-name is
                                         GlobalManagedCompute.
    --accelerator-type                 : Accelerator type (e.g., A100_80GB, H100_80GB).
                                         Only applicable when --sku-name is
                                         GlobalManagedCompute.

DeploymentScaleSettings Arguments
    --scale-type                       : Scale settings scale type.
    --scale-capacity                   : Scale settings capacity.
```

### 2.6 Examples

```
Deploy an ADM model (standard):
    az cognitiveservices account deployment create -g rg1 -n account1 \
      --deployment-name dep1 --model-format OpenAI --model-name gpt-4 \
      --model-version 0613 --sku-name Standard --sku-capacity 1

Deploy a MaaP model (GlobalManagedCompute):
    az cognitiveservices account deployment create -g rg1 -n account1 \
      --deployment-name dep1 \
      --model-id azureml://registries/reg1/models/model1/versions/1 \
      --sku-name GlobalManagedCompute --sku-capacity 4

Deploy a MaaP model with deployment template and accelerator:
    az cognitiveservices account deployment create -g rg1 -n account1 \
      --deployment-name dep1 \
      --model-id azureml://registries/reg1/models/model1/versions/1 \
      --sku-name GlobalManagedCompute --sku-capacity 4 \
      --deployment-template azureml://registries/reg1/deploymenttemplates/dt1/versions/1 \
      --accelerator-type A100_80GB
```

### 2.7 Implementation Summary

| File | Change |
|------|--------|
| `_params.py` | Add `--model-id`, `--deployment-template`, `--accelerator-type` in new `MaaP Deployment` argument group. Make `--model-format` optional. |
| `custom.py` | Add params to `deployment_begin_create_or_update()`. Wire to SDK objects. Add validation for mutual exclusivity and conditional requirements. |
| `_help.py` | Add MaaP deployment examples. |

---

## 3. Publisher – `az ml`

**Only change: bump SDK version** in `setup.py` (`azure-ai-ml`). The `az ml` commands use SDK loader functions (`load_model()`, `load_deployment_template()`) that handle all YAML parsing and field mapping. The CLI has no knowledge of individual YAML fields — new YAML fields are supported automatically once the SDK is updated.

### New YAML fields (for reference)

**`az ml deployment-template create`** — `accelerator_maps`:
```yaml
accelerator_maps:
  - accelerator_type: H100_80GB
    number_of_accelerators_per_model_instance: 4
    default: true
  - accelerator_type: H200_141GB
    number_of_accelerators_per_model_instance: 2
```

**`az ml model create`** — `allowed_deployment_templates`:
```yaml
allowed_deployment_templates:
  - asset_id: "azureml://registries/reg1/deploymenttemplates/dt1/labels/latest"
  - asset_id: "azureml://registries/reg1/deploymenttemplates/dt2/labels/latest"
  - asset_id: "azureml://registries/reg1/deploymenttemplates/dt3/versions/1"
```

---

## 4. Open Items

| Item | Owner | Blocking CLI? |
|------|-------|--------------|
| SDK: `DeploymentModel.source` for `--model-id` | SDK team | Yes |
| SDK: new fields for `--deployment-template`, `--accelerator-type` | SDK team | Yes |
| `--model-format` optional change (CLI) | Amit | No — can ship independently |
