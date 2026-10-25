# Starktastic Homelab: system overview

Cross-repo guide. It lives in the `.github` repo and is symlinked as `~/Projects/homelab/CLAUDE.md`, so it
loads for every repo under that folder; each repo's own `CLAUDE.md` has its commands and gotchas.
`profile/README.md` is the public, human-readable overview; keep it consistent with this file.

## Repos

| Repo | Does | What `main` triggers |
|---|---|---|
| `packer` | Debian 13 Proxmox template (VM 900) | push → real build (`build.yml`) |
| `terraform` | K3s VMs cloned from the template | PR merge → apply (`apply.yml`) |
| `ansible` | Proxmox host, K3s, ArgoCD bootstrap | push or dispatch → full deploy (`deploy.yml`) |
| `apps` | ArgoCD GitOps workloads | ArgoCD pulls `main`; push → refresh (`refresh.yaml`) |
| `.github` | Org profile, diagrams, this file, shared Claude settings | PR → diagram render (`diagrams.yml`) |

## Trigger chain

1. packer `build.yml` (push to `main`, `workflow_dispatch`) builds the template, publishes a release and
   opens PR `packer-build-<run>` in terraform updating `packer-manifest.json`. There is no dispatch here.
2. Someone merges that PR (or any terraform PR). terraform `apply.yml` (`pull_request_target: closed`)
   applies the saved plan and, if anything changed, sends `repository_dispatch` event
   `infrastructure-changed` with payload key `drained_nodes` to ansible.
3. ansible `deploy.yml` (`repository_dispatch: infrastructure-changed`, push, `workflow_dispatch`) runs
   `k3s.yml`, installs ArgoCD with a root app on apps `bootstrap/`, and rewrites the org secret
   `KUBECONFIG_RAW` for terraform and apps.
4. ArgoCD reconciles apps `main` through RollingSync; apps `refresh.yaml` refreshes touched apps on push.

Side channels: packer and ansible `i915-compat.yml` read each other's i915 pins from `main`; terraform
`drift.yml` (daily) and packer `check-debian-iso.yml` (weekly) open issues and PRs; ansible
`i915-sriov-upgrade.yml` reboots the Proxmox host when the host pin changes.

**Docs-only commits are live events** in packer, terraform and ansible: their `paths-ignore` lists cover
only `README.md`, `renovate.json` and a few ansible host paths. Put `[skip ci]` in the commit subject
(for terraform, in the PR title).

## Contract index

| Contract | Producer | Consumer |
|---|---|---|
| Image manifest | packer `build.pkr.hcl`, `config.pkr.hcl`, `.github/workflows/build.yml` | terraform `packer-manifest.json`, `.github/workflows/{apply,validate-and-plan,drift}.yml`, `variables.tf` |
| Apply → deploy dispatch | terraform `.github/workflows/apply.yml`, `scripts/maintenance_apply.py` | ansible `.github/workflows/deploy.yml`, `k3s.yml` |
| VM tags, SSH address | terraform `main.tf` | ansible `inventory/proxmox.yml` |
| Maintenance lock | ansible `scripts/maintenance_lock.py` | terraform `.github/workflows/{apply,validate-and-plan}.yml`, `scripts/maintenance_apply.py`; ansible `.github/workflows/deploy.yml`, `roles/k3s_node_retirement/` |
| `KUBECONFIG_RAW` | ansible `.github/workflows/deploy.yml` (`handoff` job) | terraform `.github/workflows/apply.yml`; apps `.github/workflows/refresh.yaml` |
| i915 pins and checker | packer `debian.auto.pkrvars.hcl`; ansible `group_vars/proxmox_hosts/i915_sriov.yml`, `roles/i915_sriov/defaults/main.yml` | the other repo's `.github/workflows/i915-compat.yml`; `scripts/i915_compat.py` + test, byte-identical in both |
| CSI pool, slots, topology | terraform `terraform.tfvars`, `modules/vm/main.tf`; ansible `group_vars/all/proxmox_csi.yml` | apps `infrastructure/system/proxmox-csi/`, `services/operations/kube-prometheus-stack/manifests/proxmox-csi-capacity.yaml` |
| ArgoCD root app, notifications | ansible `roles/bootstrap_cluster/tasks/argocd.yml` | apps `bootstrap/appsets/`, `infrastructure/configs/templates/argocd/` |
| Sealed-secrets key and cert | ansible `roles/bootstrap_cluster/tasks/main.yml` | apps `scripts/sealed-secrets-cert.pem`, `scripts/seal.sh`; ansible `csi-activation-preflight.yml` (pins apps by SHA) |
| Worker labels | ansible `roles/k3s_workers/tasks/main.yml` | apps `infrastructure/system/proxmox-csi/values.yaml`, `infrastructure/system/intel-device-operator/manifests/gpu-device-plugin.yaml` |
| Cert backup → restore | apps `infrastructure/base-configs/templates/cert-backup/` | ansible `roles/bootstrap_cluster/tasks/restore-certs.yml` |
| NFS server and path | apps `templates/globals.yaml`, `infrastructure/system/nfs-provisioner/values.yaml` | ansible `group_vars/all/vars.yml` |
| Diagram tooling (copies) | `.github` `scripts/render-diagrams.sh`, `scripts/check-readme-images.py`, `.github/workflows/diagrams.yml` | the same files in terraform and apps |

## Source of truth

| Facts | Owner |
|---|---|
| Template image, guest i915 pair, Debian ISO | packer |
| VM count, sizes, IPs, VMIDs, tags, CSI resource pool | terraform |
| K3s and kube-vip versions, host i915 pin, CSI slot budget and topology, lock helper, ArgoCD install | ansible |
| Workloads, storage classes and PVs, image versions | apps |
| Cross-repo overview, diagrams, shared Claude settings | .github |

NFS server and path are duplicated in apps and ansible with no single owner. TODO(ben): verify which wins.

## Shared rules

- **Runner:** every self-hosted job uses the bare `self-hosted` label. The repos describe it as amd64 VM300
  on the Proxmox host (maintenance state in `/var/lib/homelab-maintenance`), so a host reboot takes it down.
  TODO(ben): verify (it has also been described as a Raspberry Pi).
- **Secrets:** all five repos are public. Refer to secrets by name only (`ORG_DISPATCH_TOKEN`,
  `KUBECONFIG_RAW`, the per-repo Proxmox, S3 and vault names); never read or print values. Never read
  `id_rsa*`, `pve-creds.txt`, `runner-ssh.sh`, `*.key` or `.recovery/` in the parent folder, or any
  `.vault_pass`, `.env`, kubeconfig, state file or vaulted file. Keep secrets out of workflow artifacts,
  logs and the profile.
- **Apply policy, as implemented:** Packer opens a manifest PR; merging any terraform PR applies its saved
  plan with auto-approve. Branch protection requires status checks, no approvals, and admins can bypass.
  TODO(ben): verify the intended policy (Packer-triggered base-VM applies auto-approved, Terraform
  template changes need PR approval) and enforce it.
- **Live operations** (`kubectl`, `helm`, lock handling) run from the runner, not the workstation.

## Working across repos

- `~/Projects/homelab` is not a git repo: run git inside the specific repo (`git -C <repo> …`).
- Split a cross-repo change into matching per-repo commits, and list them (repo, files, message) for Ben.
- Use subagents for parallel per-repo investigation. Use plan mode for anything that spans repos or
  touches CI or dispatch wiring.
- Commit only when asked. Never push or open PRs unless asked. Never commit secrets.
- Codex leftovers (`codex/*` branches, `.worktrees/`, `~/.codex/worktrees/`, `docs/superpowers/`) are
  history. Don't delete ansible's `codex/storage-maintenance`: CI pins the lock helper to a commit that only
  that branch contains.
- Claude settings: each repo's `.claude/settings.json` and `.github/.claude/settings.json` (symlinked as
  `~/Projects/homelab/.claude`) are identical. Settings load only from the launch directory, so change all
  five together.
