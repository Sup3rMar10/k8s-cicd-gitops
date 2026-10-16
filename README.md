# 🚢 k8s-cicd-gitops

> **Reference configuration.** Written to show the pipeline; it has not been run or tested end to end. The workflows have never executed and Argo CD was never connected to a cluster. Only `kubectl kustomize` was used to check that the overlays render.

A small GitOps delivery setup for one tiny service: a GitHub Actions pipeline that builds, scans and pushes an image, a Kustomize layout for dev and prod, and Argo CD applications that deploy from Git. The point is the shape of the pipeline and why each piece is there.

Original files. `OWNER/REPO` in the manifests and in `argocd/` are placeholders you replace with your own GitHub path.

## How a change flows

```
 git push (main)
      │
      ▼
 GitHub Actions: ci.yml
   build image ──▶ Trivy scan ──▶ push to GHCR (tag = commit SHA)
                     │ fails on CRITICAL
      │
      ▼
 commit to Git: deploy/overlays/dev  newTag = <SHA>
      │
      ▼
 Argo CD "app-dev" (auto-sync, prune, selfHeal) ──▶ dev namespace

 Promotion to prod: open a PR that copies the tag into deploy/overlays/prod
      │  validate.yml renders both overlays and schema-checks them on the PR
      ▼
 merge ──▶ a person syncs Argo CD "app-prod" (manual) ──▶ prod namespace
```

## Files

```
app/                      tiny stdlib HTTP service (/healthz) and its Dockerfile (slim, non-root uid 10001)
deploy/base/              Deployment (probes, requests/limits, securityContext), Service, PodDisruptionBudget
deploy/overlays/dev/      namespace app-dev, image tag written by CI
deploy/overlays/prod/     namespace app-prod, bigger resources, HorizontalPodAutoscaler, image tag set by PR
argocd/                   Argo CD Application for dev (automated) and prod (manual sync)
.github/workflows/        ci.yml (build, scan, push, bump tag) and validate.yml (PR checks)
```

## Why it is built this way

- **Tag images with the commit SHA, never `:latest`.** A tag that names one commit is immutable in practice, so "what is running" is a fact you can look up, and a rollback is a known earlier SHA.
- **Git is the source of truth.** CI never talks to the cluster. It records the new tag in Git, and Argo CD makes the cluster match. Deploys, rollbacks and audit all become Git history.
- **Least-privilege token.** The workflow defaults to `contents: read` and the job asks only for `contents: write` (to commit the tag) and `packages: write` (to push). It uses the automatic `GITHUB_TOKEN`, so there is no long-lived credential to store.
- **Scan before push.** The image is built and loaded locally, scanned, and only pushed if no fixable CRITICAL vulnerability is found, so a failing image never reaches the registry.
- **Dev auto, prod manual.** Dev syncs on its own with `prune` and `selfHeal`. Prod has no automated sync: a merged PR changes Git, and a person decides when to sync.
- **HPA and Argo CD.** The prod app ignores `/spec/replicas` on the Deployment, otherwise `selfHeal` and the autoscaler would keep undoing each other.
- **Hardened pod spec.** Non-root, read-only root filesystem, all capabilities dropped, seccomp default, no service-account token, `maxUnavailable: 0` rolling updates, a PodDisruptionBudget.
- **Actions pinned by version tag.** Pinning by commit SHA is stricter (see below).

## What was and was not checked

- **Checked:** `kubectl kustomize` renders both overlays (dev: Deployment, Service, PDB; prod: those plus the HPA), and every YAML file parses.
- **Not checked, so treat as likely to need fixes:** nothing in `.github/workflows/` has run; the Docker image was never built; Argo CD was never pointed at this repo; the kubeconform image tag in `validate.yml` was not confirmed to exist; the assumption that `kustomize` is preinstalled on the runner is unverified; the CI commit step needs the repo to allow the workflow to push to `main` (branch protection would block it); the Trivy gate was never exercised; prod renders `replicas: 2` from the base and relies on the HPA's `minReplicas: 3` to take over.

## What I would add for a real customer

- **Sign images** with cosign (keyless) and verify the signature at admission.
- **Generate an SBOM** and attach it to the image.
- **Environment protection rules** on prod: required reviewers, and a branch protection rule so CI commits go through a PR.
- **Secrets management:** an external secrets operator or sealed secrets. This repo has no secrets at all.
- **Pin actions by commit SHA** and let Dependabot update them.
- **Rollback by Git revert**, with a documented drill, and Argo CD notifications to chat on sync failure.
- **Progressive delivery** (Argo Rollouts) once a plain rolling update is not enough.
