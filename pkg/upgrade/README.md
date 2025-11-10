# Gateway API CRD Installer

This directory contains a Kubernetes Job designed to safely install or upgrade [Gateway API](https://gateway-api.sigs.k8s.io/) Custom Resource Definitions (CRDs) in a cluster.

## Overview

Bundling Gateway API CRDs directly with add-ons (e.g., via Helm charts) can lead to conflicts when multiple controllers try to manage the same CRDs, potentially downgrading them or switching release channels unexpectedly.

This Job provides a safe mechanism to:
1.  **Install** missing Gateway API CRDs.
2.  **Upgrade** existing CRDs to a requested version *only if* the requested version is newer.
3.  **Avoid Conflicts** by refusing to overwrite CRDs from a different release channel (e.g., Standard vs. Experimental).

## Components

* **`job.yaml`**: The Kubernetes Job that runs the installation logic.
* **`rbac.yaml`**: ServiceAccount, ClusterRole, and ClusterRoleBinding allowing the Job to manage CRDs.
* **`configmap.yaml`**: Contains the `install.sh` script that implements the safe upgrade state machine.

## Configuration

The Job is configured via environment variables set in `job.yaml`:

| Environment Variable | Required | Description | Example |
| :--- | :--- | :--- | :--- |
| `GATEWAY_API_VERSION` | Yes | The target Gateway API release tag. | `v1.2.0` |
| `RELEASE_CHANNEL` | Yes | The release channel to use. Must be `standard` or `experimental`. | `standard` |
| `CRD_SUBSET` | No | A comma-separated list of specific CRDs to manage. If unset, defaults to all CRDs for the chosen channel. | `gateways.gateway.networking.k8s.io,httproutes.gateway.networking.k8s.io` |

## Usage

1.  **Apply RBAC and ConfigMap:**
    ```bash
    kubectl apply -f rbac.yaml
    kubectl apply -f configmap.yaml
    ```

2.  **Configure and Run the Job:**
    Edit `job.yaml` to set your desired `GATEWAY_API_VERSION` and `RELEASE_CHANNEL`, then apply it:
    ```bash
    kubectl apply -f job.yaml
    ```

3.  **Check Logs:**
    ```bash
    kubectl logs -l job-name=gateway-api-crd-install
    ```

## Logic Details

For every CRD processed, the installer script performs the following checks:

1.  **Existence Check:** If the CRD does not exist in the cluster, it is installed immediately from the official upstream source.
2.  **Channel Safety Check:** If the CRD exists, it checks the `gateway.networking.k8s.io/channel` label. If the installed channel does not match the requested `RELEASE_CHANNEL`, the script skips this CRD to avoid accidentally switching a cluster from Experimental to Standard (or vice-versa) potentially breaking fields.
3.  **Version Check:** It compares the requested `GATEWAY_API_VERSION` with the installed `gateway.networking.k8s.io/bundle-version` annotation.
    * If installed version < requested version: **UPGRADE**.
    * If installed version >= requested version: **SKIP** (already up-to-date).
