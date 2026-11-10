# Execution Order: OpenShift Software Factory - Agents as Staff (AWS)

## Complete Flow

### 1. Pool Provisioning (agd-v2/ocp-cluster-aws-pools)
**Cloud Provider:** AWS  
**Action:** Provisions multinode OpenShift 4.22 cluster
- 3 control plane nodes
- 3 workers (c5a.2xlarge: 8 vCPU, 16 GiB each)

**Propagates to Component:**
- `openshift_api_url` → `openshift_api_url`
- `openshift_cluster_admin_token` → `openshift_api_key`
- Bastion connection details
- Sandbox credentials

---

### 2. Component Workloads (Execute in Order)

#### a. `agnosticd.core_workloads.ocp4_workload_openshift_data_foundation`
**Purpose:** Install ODF storage  
**Channel:** stable-4.22  
**Sets:** Default storage class

---

#### b. `agnosticd.core_workloads.ocp4_workload_authentication`
**Purpose:** Install Keycloak and create users  
**Provider:** Keycloak v26.6  
**Creates:**
- Realm: `sso`
- Users: `user1`, `user2`, ..., `userN`
- KeycloakRealmImport with user passwords
- Admin user: `admin`

**Populates agnosticd_user_data:**
- `keycloak_admin_console`
- `keycloak_admin_user`
- `keycloak_admin_password`

---

#### c. `agnosticd.core_workloads.ocp4_workload_openshift_gitops`
**Purpose:** Install ArgoCD/OpenShift GitOps  
**Channel:** gitops-1.21  
**Configuration:**
- Cluster admin access
- RBAC for admin group
- Updated controller/repo resources

---

#### d. `agnosticd.core_workloads.ocp4_workload_gitops_bootstrap`
**Purpose:** Create ArgoCD Application for software factory deployment

**Creates ArgoCD Application:**
- **Name:** `software-factory-deployer`
- **Namespace:** `openshift-gitops`
- **Repo:** https://github.com/rhpds/software-factory-deployer
- **Branch:** `field-content`
- **Path:** `automation/gitops/bootstrap`

**Passes Helm Values:**
```yaml
keycloak:
  consoleUrl: <from agnosticd_user_data>
  adminUser: <from agnosticd_user_data>
  adminPassword: <from agnosticd_user_data>
litemaas:
  enabled: true
  apiUrl: <from MaaSSandbox>
  apiKey: <from MaaSSandbox>
  model: "claude-sonnet-4-5"
```

**ArgoCD Syncs → Helm Chart Creates:**
1. **Namespace:** `software-factory-deployer`
2. **Secret:** `field-content-keycloak`
   - `adminUser`
   - `adminPassword`
   - `consoleUrl`
3. **Secret:** `field-content-litemaas`
   - `apiUrl`
   - `apiKey`
   - `model`

---

#### e. `software_factory_deployer.automation.ocp4_workload_software_factory`
**Purpose:** Deploy complete software factory platform

**Collection Source:**
```yaml
- name: https://github.com/rhpds/software-factory-deployer.git#/automation/ansible
  type: git
  version: field-content
```

**Execution Flow:**

##### pre_workload.yml
- Waits for ArgoCD Application `software-factory-deployer` to reach:
  - `status.sync.status: Synced`
  - `status.health.status: Healthy`
- Timeout: 600 seconds (10 minutes)

##### workload.yml - Runs Stages in Order:
1. **operators** - Deploy operators (Agent Office, Quay, Kata, etc.)
2. **kata** - Convert one worker node to run Kata RuntimeClass
3. **gitea** - Deploy Gitea with per-user orgs and tokens
4. **keycloak_users** - Configure Keycloak realm, users, OIDC clients
5. **rhdh** - Deploy Red Hat Developer Hub with OIDC, Gitea, GitOps
6. **agent_platform** - Deploy Agent Office platform, SCC, RBAC
7. **rhoai_maas** - Configure OpenShift AI MaaS with LiteMaaS endpoint
8. **mattermost** - Deploy Mattermost chat with Gitea sign-in
9. **quay** - Deploy Quay registry, mirror skill artifacts, configure Tekton Chains
10. **seats** - Create per-user namespaces with RBAC, quotas, egress policies
11. **mcp_gateway** - Deploy MCP gateway with Gitea server (54 tools)
12. **model_desk** - Configure facilitator's Claude lane (requires GCP credentials)
13. **verify** - Verify front doors: Gitea, Developer Hub, Mattermost, Registry, MCP
14. **userinfo** - Populate user data (ConfigMap + agnosticd_user_info)

##### userinfo.yml
**Creates ConfigMap:** `software-factory-userinfo`
- Namespace: `software-factory-deployer`
- Label: `demo.redhat.com/userinfo`

**Calls agnosticd_user_info for each user:**
```yaml
data:
  password: <from KeycloakRealmImport>
  openshift_console_url: <cluster console>
  openshift_api_url: <cluster API>
  gitops_url: <ArgoCD URL>
  gitea_url: <Gitea URL>
  devhub_url: <Developer Hub URL>
  mattermost_url: <Mattermost URL>
  registry_url: <Quay URL>
  artifact_registry: <Quay host>
  registry_org: "workshop"
  ocp_version: <OCP minor version>
  cluster_domain: <cluster domain>
```

**NOTE:** `showroom_url` is NOT set here - the official showroom role handles that

---

#### f. `agnosticd.showroom.ocp4_workload_showroom`
**Purpose:** Deploy per-user Showroom lab guides

**Collection:** showroom v1.8.0

**Configuration:**
```yaml
ocp4_workload_showroom_content_git_repo: https://github.com/rhpds/software-factory-deployer.git
ocp4_workload_showroom_content_git_repo_ref: field-content
ocp4_workload_showroom_content_antora_playbook: site.yml
ocp4_workload_showroom_terminal_type: showroom
```

**Reads:** agnosticd_user_info (password, URLs, cluster info)  
**Writes:** agnosticd_user_info (adds `showroom_url` for each user)  
**Deploys:** Per-user Showroom pod with lab guide content from `content/` directory

---

## Key Data Flows

### 1. User Passwords
```
ocp4_workload_authentication
  → Creates KeycloakRealmImport with user passwords
    → software_factory: keycloak_users stage reads passwords
      → software_factory: userinfo stage calls agnosticd_user_info with passwords
        → showroom role reads passwords from agnosticd_user_info
```

### 2. Keycloak Admin Credentials
```
ocp4_workload_authentication
  → Populates agnosticd_user_data
    → ocp4_workload_gitops_bootstrap reads from agnosticd_user_data
      → Passes to Helm chart values
        → Helm creates Secret field-content-keycloak
          → software_factory: keycloak_users stage reads Secret
```

### 3. LiteMaaS Credentials
```
MaaSSandbox sandbox
  → maas_sandbox.credentials
    → ocp4_workload_gitops_bootstrap passes to Helm
      → Helm creates Secret field-content-litemaas
        → software_factory: rhoai_maas stage reads Secret
```

### 4. Showroom URLs
```
software_factory: userinfo stage
  → Calls agnosticd_user_info (no showroom_url yet)
    → showroom role deploys per-user guides
      → showroom role calls agnosticd_user_info (adds showroom_url)
```

---

## Critical Wait Points

### Wait #1: GitOps Application Sync
**Where:** `software_factory_deployer.automation.ocp4_workload_software_factory` pre_workload  
**What:** ArgoCD Application `software-factory-deployer` must be Synced + Healthy  
**Why:** Secrets must exist before stages run  
**Timeout:** 600 seconds

### Wait #2: Kata Node Reboot
**Where:** software_factory kata stage  
**What:** Worker node reboots after KataConfig applied  
**Duration:** ~25 minutes  
**Why:** Must complete before workloads deploy

### Wait #3: Quay Registry Ready
**Where:** software_factory quay stage  
**What:** QuayRegistry endpoint must be available  
**Timeout:** 1800 seconds (30 minutes)

---

## Variables Reference

### From Pool to Component
- `openshift_api_url`
- `openshift_api_key` / `openshift_cluster_admin_token`
- `openshift_cluster_ingress_domain` → `cluster_domain`

### From MaaSSandbox
- `maas_sandbox.credentials[].api_base_url` → `litellm_api_base_url`
- `maas_sandbox.credentials[].virtual_key` → `litellm_virtual_key`
- `maas_sandbox.credentials[].available_models` → `litellm_available_models`

### From Authentication Workload
- `keycloak_admin_console` (agnosticd_user_data)
- `keycloak_admin_user` (agnosticd_user_data)
- `keycloak_admin_password` (agnosticd_user_data)

---

## Removed from Field-Content Branch

### Deleted Files/Directories
- `automation/ansible/run_local.yml` - Job playbook (no longer needed)
- `automation/ansible/roles/deploy-showroom/` - Old showroom role
- `automation/ansible/roles/ocp4_workload_software_factory_showroom/` - Bridge role
- `automation/ansible/roles/ocp4_workload_software_factory/tasks/showroom.yml`
- `automation/ansible/roles/ocp4_workload_software_factory/tasks/showroom_seat.yml`

### Removed from Helm Chart
- All Job configuration (image, resources, serviceAccount, rbac)
- Ansible repository settings
- Collection requirements
- Extra vars configuration

### Simplified Helm Chart
**Purpose:** Create namespace + secrets only  
**Version:** 0.2.0  
**Contents:**
- Namespace: `software-factory-deployer`
- Secret injection points: keycloak, litemaas

---

## Success Verification

### After software_factory role completes:
- ✅ All operators installed
- ✅ Gitea running with user orgs
- ✅ Developer Hub accessible with OIDC
- ✅ Mattermost running with Gitea sign-in
- ✅ Quay registry with mirrored artifacts
- ✅ MCP gateway with 54 Gitea tools
- ✅ Per-user namespaces with RBAC + quotas
- ✅ agnosticd_user_info populated (no showroom_url yet)
- ✅ ConfigMap `software-factory-userinfo` created

### After showroom role completes:
- ✅ Per-user Showroom pods deployed
- ✅ Lab guide content from `content/` directory
- ✅ agnosticd_user_info updated with `showroom_url`
- ✅ Users can access their lab guides

---

## Troubleshooting

### GitOps Application not syncing?
- Check: `oc get application software-factory-deployer -n openshift-gitops -o yaml`
- Look for: `status.sync.status` and `status.health.status`
- Check logs: `oc logs -n openshift-gitops -l app.kubernetes.io/name=argocd-application-controller`

### Secrets not created?
- Check Helm release: `helm list -n openshift-gitops`
- Check secrets: `oc get secrets -n software-factory-deployer`
- Expected: `field-content-keycloak`, `field-content-litemaas`

### Software factory stage fails?
- Check logs in deployer pod
- Verify secrets exist before role runs
- Check that GitOps Application is Healthy

### Showroom not deploying?
- Check that agnosticd_user_info was populated by software_factory role
- Verify showroom collection version is v1.8.0
- Check showroom content repo/branch are accessible

