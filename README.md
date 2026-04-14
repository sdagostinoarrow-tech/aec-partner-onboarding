# aec-partner-onboarding

Ansible role and wrapper playbooks for onboarding new partner tenants to
the Arrow Experience Center (AEC) shared OpenShift demo cluster.

A single playbook run produces:

- A new OpenShift namespace with admin RBAC, resource quota, limit ranges,
  and network policies for tenant isolation.
- htpasswd users and an OpenShift group, scoped to that namespace.
- OperatorHub access so partners can install namespace-scoped operators.
- An AAP Controller organization, team, users, OpenShift credential, and
  starter inventory.
- AAP EDA users with the Contributor role.
- A RHOAI Data Science Project with a MinIO data connection, per-user
  workbenches, an optional shared Gaudi workbench, and a Data Science
  Pipelines Application (DSPA).
- A credentials handoff file containing everything the partner needs.

## Repo layout

```
aec-partner-onboarding/
├── README.md
├── ansible.cfg
├── requirements.yml
├── inventory/hosts.yml
├── playbooks/
│   ├── onboard-partner.yml          # main entry point
│   └── offboard-partner.yml         # stub (not implemented)
└── roles/partner_onboarding/
    ├── defaults/main.yml            # all variables and toggles
    ├── vars/quota_{small,medium,large}.yml
    ├── templates/credentials.txt.j2
    └── tasks/
        ├── main.yml                 # orchestrator
        ├── 10_namespace.yml         # Phase 1
        ├── 20_identity.yml          # Phase 2
        ├── 30_operator_access.yml   # Phase 3
        ├── 40_aap_controller.yml    # Phase 4
        ├── 50_aap_eda.yml           # Phase 5
        ├── 60_rhoai_connection.yml  # Phase 6a
        ├── 61_rhoai_workbenches.yml # Phase 6b
        ├── 62_rhoai_dspa.yml        # Phase 6c
        └── 90_handoff.yml           # Phase 7
```

## Prerequisites

### On the cluster

- OpenShift 4.18+ with admin access for the human running the role
- htpasswd identity provider already configured (the role only edits the
  existing `htpasswd-secret` in `openshift-config`, it does not create
  the IdP itself)
- RHOAI installed cluster-wide (`redhat-ods-operator`)
- AAP installed cluster-wide (Controller + EDA, split deployment is fine)
- External MinIO with admin credentials in a Secret somewhere (defaults to
  `minio-storage` in `ai-team` — adjust via `rhoai_minio_admin_secret_*`)
- A `cluster-config-reader` ClusterRole — created automatically on first
  run if it doesn't exist

### In AAP

- A Project pointing at this repo (SCM type Git, your GitHub URL)
- A Custom Credential Type that injects the four admin env vars (see below)
- A Credential of that type holding cluster-admin Controller and EDA passwords
- An OpenShift credential for `oc` API calls (the bearer token type works)
- A Job Template wired up to `playbooks/onboard-partner.yml` with a survey

### Custom Credential Type for AAP admin creds

Create this once in AAP under Administration → Credential Types:

**Name:** `AEC Cluster Admin`

**Input configuration:**

```yaml
fields:
  - id: controller_username
    type: string
    label: Controller Admin Username
  - id: controller_password
    type: string
    label: Controller Admin Password
    secret: true
  - id: controller_eda_username
    type: string
    label: EDA Admin Username
  - id: controller_eda_password
    type: string
    label: EDA Admin Password
    secret: true
required:
  - controller_username
  - controller_password
  - controller_eda_username
  - controller_eda_password
```

**Injector configuration:**

```yaml
env:
  CONTROLLER_USERNAME: '{{ controller_username }}'
  CONTROLLER_PASSWORD: '{{ controller_password }}'
  CONTROLLER_EDA_USERNAME: '{{ controller_eda_username }}'
  CONTROLLER_EDA_PASSWORD: '{{ controller_eda_password }}'
```

Then create one Credential of this type holding your admin passwords and
attach it to the onboarding Job Template.

## Survey

Set up the Job Template survey with these fields:

| Prompt | Variable | Type | Required | Default |
|---|---|---|---|---|
| Partner name | `partner_name` | Text | yes | — |
| Namespace | `partner_namespace` | Text | yes | — |
| Description | `partner_description` | Textarea | no | — |
| Channel manager | `partner_channel_manager` | Text | no | — |
| Expiration date (YYYY-MM-DD) | `partner_expiration_date` | Text | yes | — |
| Quota profile | `quota_profile` | Multiple Choice | yes | medium |
| Gaudi quota count | `gaudi_quota_count` | Integer | no | 2 |
| Users (YAML list) | `partner_users` | Textarea | yes | (see below) |
| Enable operator access | `enable_operator_access` | Multiple Choice | no | true |
| Enable AAP Controller | `enable_aap_controller` | Multiple Choice | no | true |
| Enable AAP EDA | `enable_aap_eda` | Multiple Choice | no | true |
| Enable RHOAI | `enable_rhoai` | Multiple Choice | no | true |
| Enable DSPA | `enable_rhoai_dspa` | Multiple Choice | no | true |
| Enable shared Gaudi workbench | `enable_rhoai_shared_gaudi_workbench` | Multiple Choice | no | true |

The `partner_users` survey field is a textarea that accepts YAML. Help
text should show this example:

```yaml
- username: jane.doe
  email: jane.doe@example.com
  first_name: Jane
  last_name: Doe
- username: john.smith
  email: john.smith@example.com
  first_name: John
  last_name: Smith
```

## Running from a laptop

Useful for development and dry runs.

```bash
# Install collections
ansible-galaxy collection install -r requirements.yml

# Set admin credentials in your environment
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD='...'
export CONTROLLER_EDA_USERNAME=admin
export CONTROLLER_EDA_PASSWORD='...'

# Make sure you're logged in to the cluster
oc login --server=https://api.ocp.rhat.aessatl.arrow.com:6443

# Create a vars file for the partner
cat > vars/partner-example.yml <<'EOF'
partner_name: ExampleCo
partner_namespace: exampleco
partner_description: Example partner for OCPV migration POC
partner_channel_manager: someone@arrow.com
partner_expiration_date: "2026-12-31"
quota_profile: medium
gaudi_quota_count: 2
partner_users:
  - username: alice.example
    email: alice@example.com
    first_name: Alice
    last_name: Example
EOF

# Run it
ansible-playbook playbooks/onboard-partner.yml -e @vars/partner-example.yml
```

The credentials file lands at `handoffs/exampleco-credentials.txt`.

## Idempotence: read this before re-running

The role is **safe to re-run** but with one important caveat:

- Provisioning operations (namespace, RBAC, quota, users, AAP org/team,
  workbenches, DSPA) are idempotent. A re-run against an existing partner
  is a no-op for the things that already exist.
- **Passwords are NOT updated on re-run.** All user-creation modules use
  `update_secrets: false`, so the actual stored passwords stay the same.
- **The credentials file output, however, IS regenerated on every run** —
  with new randomly-generated passwords that DO NOT MATCH what's actually
  in the system. The file is wrong on a re-run.
- **Treat the first run as the source of truth** for credentials. Save
  the credentials file somewhere safe immediately after the first run.
- If you need to rotate passwords, that requires a separate playbook
  (not yet written) that explicitly opts in to `update_secrets: true`.

A future improvement: detect existing partners and either skip credential
file generation, or read existing creds from a state store. For now,
discipline.

## Known issues and workarounds

### RHOAI 3.2.0 webhook panic on workbench updates

Symptom: After the DSPA exists in a partner namespace, any attempt to
stop/start/edit a workbench through the dashboard or via `oc patch` fails
with:

```
admission webhook "notebooks.opendatahub.io" denied the request: panic:
runtime error: invalid memory address or nil pointer dereference [recovered]
```

Cause: `extractElyraRuntimeConfigInfo` in the notebook controller dereferences
a nil pointer when reconciling Notebook CRs in namespaces with a DSPA. The
`notebooks.opendatahub.io/inject-elyra-secret: "false"` annotation is
documented as a workaround but is **ignored** by the buggy code path.

**The role mitigates this by ordering**: Phase 6b (workbenches) runs
before Phase 6c (DSPA). The initial create-and-reconcile happens cleanly
because the DSPA doesn't exist yet. Existing workbenches keep running
fine after the DSPA is added.

**The role does NOT fix the underlying bug.** Once the DSPA exists,
subsequent stop/start of workbenches via the dashboard fails. To work
around this manually:

1. Scale `rhods-operator` deployment in `redhat-ods-operator` to 0
2. Patch failure policy of `notebooks.opendatahub.io` (in the
   `odh-notebook-controller-mutating-webhook-configuration` MutatingWebhookConfiguration)
   to `Ignore`
3. Also patch failure policy of every other webhook served by
   `rhods-operator-service` to `Ignore` (there are several)
4. Patch the Notebook CR's `kubeflow-resource-stopped` annotation
5. Scale `rhods-operator` back to 3 — it will reconcile webhooks back

A proper fix requires upgrading RHOAI past the bug. **File a Red Hat
support case** with the stack trace if you haven't already.

### EDA on AAP 2.6 (split deployment) — limited automation

Phase 5 only creates EDA users. There's no organizational scoping in this
EDA version (no Orgs/Teams in the UI), and the `ansible.eda` collection
v2.11.0 has known UUID-handling bugs in modules other than user creation.
We work around the user-create bug by passing `roles: [Contributor]` even
though the doc says it's ignored on AAP 2.5+.

When you upgrade AAP to the unified gateway, EDA inherits Controller's
RBAC and Phase 5 can be expanded to mirror Phase 4.

### `update_secrets: false` and password rotation

See "Idempotence" above. Re-running the playbook does NOT rotate passwords.
A future `rotate-passwords.yml` playbook is needed for that use case.

## Troubleshooting

### "admission webhook denied the request: ... panic"

You're hitting the RHOAI webhook bug. See above.

### "ObjectStoreAvailable: False" on the DSPA

Phase 6c includes a force-reconcile step to handle the most common cause
(initial timing race). If the DSPA stays unhealthy after that, check that
the `allow-rhoai-operators` NetworkPolicy from Phase 1 was created — the
operator in `redhat-ods-applications` needs ingress access to the partner
namespace's `minio-dspa` pod for health checks.

### Workbench shows 504 Gateway Timeout in browser

The OAuth proxy sidecar can't reach the Kube API server to do TokenReview.
Verify the `allow-kube-apiserver` NetworkPolicy from Phase 1 exists. Without
it, workbenches start successfully (the pod is healthy) but the dashboard
"Open" link returns 504.

### "Restricted access — you don't have access to this section"

Partner is missing one of the operator-access ClusterRoleBindings. Confirm
all four CRBs from Phase 3 exist for the partner's group:

```bash
oc get clusterrolebinding | grep <partner_namespace>
```

Should show four lines (`packagemanifests-view`, `olm-view`,
`operatorhub-config-reader`, `cluster-config-reader`).

### "context deadline exceeded" calling AAP API

Phase 4 preflight check (uri call to `/api/v2/ping/`) failed. Verify
`aap_controller_url` is correct, and that the AAP execution environment
pod can reach it. From inside the cluster, the EE hits AAP's external
route, which hairpins through the openshift-ingress router back to the
AAP service.

### Credentials file has wrong passwords after re-run

See "Idempotence" above. Use the credentials from the first run.

## Future work

- `playbooks/offboard-partner.yml` — currently a stub
- `playbooks/rotate-passwords.yml` — explicit credential rotation
- Migrate Phase 5 to full EDA RBAC once on AAP unified gateway
- File Red Hat support case for the RHOAI webhook panic
- Replace Notebook CR YAML with a Jinja template to deduplicate per-user
  vs shared-gaudi spec
