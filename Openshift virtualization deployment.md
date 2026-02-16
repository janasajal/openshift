# OpenShift Virtualization Deployment Guide

**Author:** Sajal Jana

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Step 1: Deploy OpenShift Virtualization Operator](#step-1-deploy-openshift-virtualization-operator)
3. [Step 2: Configure HyperConverged Custom Resource](#step-2-configure-hyperconverged-custom-resource)
4. [Verification](#verification)

---

## Prerequisites

- OpenShift 4.12+ cluster with cluster-admin access
- Worker nodes with virtualization capabilities enabled
- Sufficient resources (minimum 8GB RAM, 4 vCPUs per worker node)

---
## If you are using Redhat Lab, then run this command.
lab start virtualization-deploy

## Step 1: Deploy OpenShift Virtualization Operator

1. Login to **OpenShift Web Console**

2. Create the namespace:
   - Navigate to **Home** → **Projects**
   - Click **Create Project**
   - Name: `openshift-cnv`
   - Click **Create**

3. Install the operator:
   - Navigate to **Operators** → **OperatorHub**
   - Search for `OpenShift Virtualization`
   - Click the **OpenShift Virtualization** tile
   - Click **Install**

4. Configure installation:
   - **Update Channel:** `stable`
   - **Installation Mode:** `A specific namespace on the cluster`
   - **Installed Namespace:** `openshift-cnv`
   - **Update Approval:** `Automatic`
   - Click **Install**

5. Wait for installation to complete (Status: `Succeeded`)
   - Navigate to **Operators** → **Installed Operators**
   - Verify **OpenShift Virtualization** shows `Succeeded` in `openshift-cnv` project

![Screenshot: Installed Operators showing OpenShift Virtualization]

---

## Step 2: Configure HyperConverged Custom Resource

1. Access the operator:
   - Navigate to **Operators** → **Installed Operators**
   - Select project: `openshift-cnv`
   - Click **OpenShift Virtualization**

2. Create HyperConverged instance:
   - Click the **HyperConverged** tab
   - Click **Create HyperConverged**

3. Configure the resource:
   - **Name:** `kubevirt-hyperconverged`
   - **Namespace:** `openshift-cnv` (pre-populated)
   - Leave all other settings as default
   - Click **Create**

![Screenshot: HyperConverged CR creation form]

4. Wait for deployment (5-10 minutes)
   - Monitor pod deployment in **Workloads** → **Pods** (project: `openshift-cnv`)

---

## Verification

1. Check HyperConverged status:
   - Navigate to **Operators** → **Installed Operators** → **OpenShift Virtualization**
   - Click **HyperConverged** tab
   - Verify `kubevirt-hyperconverged` shows **Phase:** `Deployed`

2. Verify all pods are running:
   - Navigate to **Workloads** → **Pods**
   - Project: `openshift-cnv`
   - All pods should show **Status:** `Running`

3. Confirm Virtualization menu:
   - Verify **Virtualization** appears in the left navigation menu
   - This indicates OpenShift Virtualization is ready

![Screenshot: Virtualization menu in console]

---

**Deployment Complete.** You can now create and manage virtual machines.
