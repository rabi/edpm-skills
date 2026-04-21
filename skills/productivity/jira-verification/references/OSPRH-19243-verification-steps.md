# Verification Steps for OSPRH-19243

**Epic:** [OSPRH-19243](https://redhat.atlassian.net/browse/OSPRH-19243) — Implement service/container cleanup after they're dropped from nodeset/deployment

## Background

This epic implements service/container cleanup automation for EDPM. When a service/container has been deployed but is later removed from the nodeset or deployment (during update/upgrade/redeployment), it needs to be cleaned up. The implementation consists of two main features:

1. **Config directories consistency** — Standardizes config file locations across services
2. **Service cleanup automation** — New `edpm_cleanup` role with state file tracking

## Pre-requisites

A deployed RHOSO environment with EDPM compute nodes (including telemetry service)


---

## 1. Verify State File Creation

**Objective:** Confirm that `edpm_container_standalone` creates and maintains the deployed services state file.

**Steps:**

1. Deploy a nodeset with services (e.g., ovn, neutron-metadata, telemetry, nova).
2. SSH into the EDPM compute node.
3. Check the state file exists:
   ```bash
   cat /var/lib/edpm-config/deployed_services.yaml
   ```
4. **Expected:** The file should contain a YAML structure like:
   ```yaml
   services:
     ovn:
       containers:
         - ovn_controller
       updated_at: <timestamp>
     neutron_metadata:
       containers:
         - ovn_metadata_agent
       updated_at: <timestamp>
     nova:
       containers:
         - nova_compute
         - nova_compute_init
       updated_at: <timestamp>
   ```
5. Verify each deployed service has an entry with its container names and a valid timestamp.

---

## 2. Verify Config Directory Consistency

**Objective:** Confirm that config files are in the new standardized locations.

**Steps:**

1. On the compute node, verify startup configs are organized per service:
   ```bash
   ls /var/lib/edpm-config/container-startup-config/
   ```
   **Expected:** One directory per service (e.g., `ovn/`, `neutron_metadata/`, `nova/`, `telemetry/`, etc.)

2. Verify each service directory contains container definition JSON files:
   ```bash
   ls /var/lib/edpm-config/container-startup-config/ovn/
   ```
   **Expected:** `ovn_controller.json`

3. Verify kolla config files exist:
   ```bash
   ls /var/lib/kolla/config_files/
   ```
   **Expected:** Per-container JSON config files (e.g., `ovn_controller.json`, `nova_compute.json`)

---

## 3. Verify Service Cleanup — Remove a Service from Nodeset

**Objective:** Confirm that removing a service from the nodeset's service list cleans up its containers, config files, and state.

**Steps:**

1. Remove a service (e.g., `telemetry`) from the `OpenStackDataPlaneNodeSet` spec by dropping it from the `services` list.

2. Create/trigger a deployment with the `cleanup` service added to `servicesOverride` (or add `cleanup` to the nodeset services list).

3. After the deployment completes, verify on the compute node:

   a. **Containers removed:**
   ```bash
   podman ps -a --format '{{.Names}}' | grep -i ceilometer
   podman ps -a --format '{{.Names}}' | grep -i node_exporter
   ```
   **Expected:** No containers for the removed service.

   b. **Startup config removed:**
   ```bash
   ls /var/lib/edpm-config/container-startup-config/telemetry/ 2>/dev/null
   ```
   **Expected:** Directory should not exist.

   c. **Kolla config removed:**
   ```bash
   ls /var/lib/kolla/config_files/ | grep -i ceilometer
   ```
   **Expected:** No kolla config files for removed service containers.

   d. **State file updated:**
   ```bash
   cat /var/lib/edpm-config/deployed_services.yaml
   ```
   **Expected:** The removed service should no longer appear in the state file.

   e. **Generic paths cleaned up:**
   ```bash
   ls /var/lib/openstack/telemetry 2>/dev/null
   ls /var/lib/openstack/healthchecks/telemetry 2>/dev/null
   ```
   **Expected:** Directories should not exist.

---

## 4. Verify Orphaned Container Cleanup (Container Dropped Within a Service)

**Objective:** When a service update removes one of its containers (e.g., disabling `nova_nvme_cleaner`), the orphaned container should be cleaned up.

**Steps:**

1. Deploy nova with the nvme cleaner enabled by setting `edpm_nova_enable_nvme_cleaner: true`. Verify the container is running:
   ```bash
   podman ps --format '{{.Names}}' | grep nova_nvme_cleaner
   ```
2. Redeploy nova with `edpm_nova_enable_nvme_cleaner: false` (the default). The nova role automatically removes the `nova_nvme_cleaner` container when this variable is false.
3. Verify the container has been removed:
   ```bash
   podman ps -a --format '{{.Names}}' | grep nova_nvme_cleaner
   ```
   **Expected:** No `nova_nvme_cleaner` container should exist.

4. Verify the state file no longer lists `nova_nvme_cleaner` under the nova service:
   ```bash
   cat /var/lib/edpm-config/deployed_services.yaml
   ```
   **Expected:** The nova service entry should only contain `nova_compute` and `nova_compute_init`.

---

## 5. Verify Image and Volume Pruning (This may not be in the version you're using)

**Objective:** After cleanup, unused container images and volumes are pruned.

**Steps:**

1. Note unused images before cleanup:
   ```bash
   podman images
   podman volume ls
   ```
2. Remove a service from the nodeset and trigger a cleanup deployment via the operator.
3. Verify unused images and volumes are removed:
   ```bash
   podman images
   podman volume ls
   ```
   **Expected:** Dangling/unused images and volumes related to removed services should be pruned.

---

## 6. Verify Idempotency

**Objective:** Running cleanup twice with the same services list should have no side effects on the second run.

**Steps:**

1. Trigger a cleanup deployment via the operator (with `cleanup` in `servicesOverride`).
2. After it completes, trigger the same cleanup deployment again.
3. **Expected:** Second run should report no changes (all tasks show `ok`, no `changed`).

---

## 7. Verify Update Path (Config Migration from FR4 to FR5)

**Objective:** During an update from FR4 to FR5, existing config files are migrated from the old layout to the new standardized locations.

**Steps:**

1. Start with a RHOSO FR4 deployment with services running on EDPM compute nodes.
2. Perform the update to FR5.
3. After the update completes, verify on the compute node:
   - Old startup config format (flat JSON files) should be migrated to the new per-service directory structure under `/var/lib/edpm-config/container-startup-config/<service>/`
   - Old kolla config files should be moved to the standardized locations under `/var/lib/kolla/config_files/`
   - The state file `/var/lib/edpm-config/deployed_services.yaml` should be created with all deployed services
4. Verify containers restart correctly with configs from the new locations.

---

## 10. Negative Test — Missing State File

**Objective:** Verify cleanup fails gracefully when the state file does not exist.

**Steps:**

1. On a fresh node that has never had services deployed (no `/var/lib/edpm-config/deployed_services.yaml`), trigger a cleanup deployment via the operator.
2. **Expected:** The ansible job fails with clear error message: "State file ... not found. Cannot proceed with cleanup."
