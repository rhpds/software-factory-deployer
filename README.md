= software-factory-deployer

AgnosticD-v2-style workload that turns a fresh OpenShift cluster into the environment for the
"OpenShift Software Factory — Agents as Staff" Showroom workshop.

Reference implementation: the `agent-office` platform (operator + Developer Hub golden paths +
MCP gateway governance + OpenShift AI). This repo re-packages that stack as a repeatable,
per-order install for RHDP.

== Repository Structure

This repository is structured as an Ansible collection monorepo:

* `automation/ansible/` — Ansible collection (`software_factory_deployer.automation`)
  ** `roles/ocp4_workload_software_factory/` — Main workload role
  ** `galaxy.yml` — Collection metadata
* `automation/gitops/bootstrap/` — Helm chart for GitOps bootstrap (creates namespace + secrets)
* `content/` — Showroom lab guide content (Antora modules)
* `site.yml` — Antora playbook for building lab guides

== Model Access

`model_access: maas` — attendees' agents consume models through OpenShift AI
**Models-as-a-Service** (early access in current OpenShift AI; the lab labels it with its real
support status). External frontier models are brokered by MaaS (keys/quotas issued per user);
an optional in-cluster vLLM registers into the same MaaS as the sovereign endpoint for the
"swap the brain" module. Agents never hold provider credentials either way.

== How It Works

This workload is deployed as part of an AgnosticV catalog item that:

1. **Provisions infrastructure** via AWS pool (multinode OpenShift cluster)
2. **Installs prerequisites** via AgnosticV workloads:
   - ODF storage
   - Keycloak authentication
   - OpenShift GitOps
3. **Bootstraps GitOps** via `ocp4_workload_gitops_bootstrap`:
   - Creates ArgoCD Application pointing to `automation/gitops/bootstrap/`
   - Helm chart creates namespace and secrets (Keycloak admin, LiteMaaS credentials)
4. **Deploys software factory** via `software_factory_deployer.automation.ocp4_workload_software_factory`:
   - Waits for GitOps Application to sync
   - Runs deployment stages (operators, gitea, Developer Hub, Agent Office, etc.)
   - Populates user data via `agnosticd_user_info` plugin
5. **Deploys lab guides** via `agnosticd.showroom.ocp4_workload_showroom`:
   - Reads user data from previous step
   - Deploys per-user Showroom pods with content from `content/` directory

== Deployment Stages

The `ocp4_workload_software_factory` role runs these stages in order:

1. **operators** — Deploy operators (Agent Office, Quay, Kata, etc.)
2. **kata** — Convert one worker node to run Kata RuntimeClass
3. **gitea** — Deploy Gitea with per-user orgs and tokens
4. **keycloak_users** — Configure Keycloak realm, users, OIDC clients
5. **rhdh** — Deploy Red Hat Developer Hub with OIDC, Gitea, GitOps
6. **agent_platform** — Deploy Agent Office platform, SCC, RBAC
7. **rhoai_maas** — Configure OpenShift AI MaaS with LiteMaaS endpoint
8. **mattermost** — Deploy Mattermost chat with Gitea sign-in
9. **quay** — Deploy Quay registry, mirror skill artifacts, configure Tekton Chains
10. **seats** — Create per-user namespaces with RBAC, quotas, egress policies
11. **mcp_gateway** — Deploy MCP gateway with Gitea server (54 tools)
12. **model_desk** — Configure facilitator's Claude lane (requires GCP credentials)
13. **verify** — Verify front doors: Gitea, Developer Hub, Mattermost, Registry, MCP
14. **userinfo** — Populate user data (ConfigMap + agnosticd_user_info)

Stage selection is controlled by `software_factory_stages` variable
(see `automation/ansible/roles/ocp4_workload_software_factory/defaults/main.yml`).

== AgnosticV Catalog Item

See `agd_v2/ocp-software-factory-agents-aws/` in the agnosticv repository for the complete
catalog item configuration.

== Development

For local iteration, you can run the role directly against a cluster:

[source,bash]
----
# Install the collection
ansible-galaxy collection install automation/ansible/

# Run against current KUBECONFIG
ansible-playbook -c local -i localhost, \
  -e ACTION=provision \
  -e num_users=2 \
  -e cluster_domain=apps.example.com \
  -e '<other required vars>' \
  -m software_factory_deployer.automation.ocp4_workload_software_factory
----

For full deployment flow, order through RHDP catalog.

== Documentation

- `EXECUTION_ORDER.md` — Complete execution flow, data flows, and troubleshooting
- `automation/gitops/bootstrap/README.md` — GitOps bootstrap Helm chart details
