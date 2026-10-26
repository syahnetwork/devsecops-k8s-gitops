# GitOps Multi-Environment Deployment — Post-Mortem & Improvement Plan

This repository contains Kubernetes manifests (`k8s/`) and a GitHub Actions GitOps pipeline
(`.github/workflows/multi1.yaml`) that deploys to **staging** and, after manual approval, to
**production**. Below is a retrospective of what went wrong during development/CI and how each
issue was (or should be) fixed, followed by a prioritized list of improvements.

---

## What Went Wrong — and the Fix

1. **Deployment workflow silently skipped / not discovered**
   - *Problem:* A stray file named `01Fully Automated Multi-Environment Deployment (GitOps-Driven)` sat inside `.github/workflows/` without a `.yaml`/`.yml` extension, so GitHub Actions ignored it — and its contents were actually a copy of the Namespace manifest, not a workflow.
   - *Fix:* Remove the misnamed/orphaned file; keep exactly one valid workflow per pipeline in `.github/workflows/*.yaml` (now `multi1.yaml`).

2. **`kubeconform` download failed in the validate step**
   - *Problem:* The workflow piped `curl .../releases/latest/download/kubeconform-linux-amd64.tar.gz | tar xz`, but kubeconform's latest release asset is a versioned URL (e.g. `.../download/v0.6.7/kubeconform-linux-amd64.tar.gz`); "latest/download" returns a 404 HTML page, `tar` then fails or produces no binary, and the job dies with a confusing error.
   - *Fix:* Pin the release version and verify the binary:
     ```bash
     VER=v0.6.7
     curl -sSL "https://github.com/yannh/kubeconform/releases/download/${VER}/kubeconform-linux-amd64.tar.gz" | tar xz
     ./kubeconform -strict -summary k8s/
     ```

3. **Port mismatch: container port 80 vs. Service targetPort 8080**
   - *Problem:* `deployment.yaml` declared `containerPort: 80` while `service.yaml` routed `targetPort: 8080`, and the ConfigMap advertises `PORT: "8080"`. Pods would receive no traffic (or nginx would listen on the wrong port), causing empty endpoints and rollout timeouts.
   - *Fix:* Make ports consistent everywhere — container listens on 8080 (`containerPort: 8080`), Service uses `targetPort: 8080`, and the Ingress still fronts `port: 80` on the Service.

4. **Liveness/readiness probes commented out → rollout hangs or flaky health**
   - *Problem:* Probes pointed at `/health` on port 8080 but were commented out in `deployment.yaml`; combined with issue #3 this produced pods that were never validated as healthy, and `kubectl rollout status` timed out at unpredictable moments.
   - *Fix:* Re-enable the probes and make sure they match the actual app port/path (nginx serves `/` — use `path: /` until the app exposes a real `/health`).

5. **`IMAGE_TAG_PLACEHOLDER` replacement was a no-op**
   - *Problem:* Both deploy jobs ran `sed -i "s|IMAGE_TAG_PLACEHOLDER|${GITHUB_SHA::7}|g" k8s/deployment.yaml`, but the image in the manifest is hard-coded as `nginx:alpine` — there is no placeholder string, so the sed matched nothing and the CI "image tag" was never actually applied. Deployments shipped an untagged/moving image.
   - *Fix:* Either put `image: nginx:IMAGE_TAG_PLACEHOLDER` in the manifest so the sed works deterministically, or better, replace sed with a declarative tool like `kustomize edit set image`.

6. **Kind cluster created but never used / conflicting with KUBECONFIG secret**
   - *Problem:* Each deploy job both configured `~/.kube/config` from `secrets.KUBECONFIG` **and** ran `helm/kind-action` to create a local `ci-cluster`. Kind overwrites the kubeconfig context, so manifests were applied to an ephemeral in-runner cluster that is destroyed when the job ends — production was never really deployed.
   - *Fix:* Pick one target: for real deployments remove `kind-action` and point at the cluster from the `KUBECONFIG` secret; keep Kind only for a separate CI smoke-test job.

7. **Same `KUBECONFIG` secret used for staging and production**
   - *Problem:* Both environments read `${{ secrets.KUBECONFIG }}`, meaning staging and production share one cluster/credential — a staging compromise reaches production, and namespace separation (`--namespace staging` vs `production`) is the only guard.
   - *Fix:* Use environment-scoped secrets (`secrets.KUBECONFIG` defined under the `staging` and `production` GitHub Environments respectively) so each job gets its own credentials.

8. **Plaintext secret committed to the repo**
   - *Problem:* `k8s/secret.yaml` contains base64-encoded (not encrypted) `DATABASE_URL` with `user:pass` and an `API_KEY` ("replace-me-with-a-real-key") checked into Git — base64 is trivially reversible and history retains it forever.
   - *Fix:* Remove real values from the repo: commit only a `.example` placeholder, inject real secrets via sealed-secrets/SOPS or the CI secret store, and rotate any credential that was exposed.

9. **`--validate=false` masking manifest errors + missing namespace on some resources**
   - *Problem:* Staging applied with `kubectl apply -f k8s/ --namespace staging --validate=false`, disabling server validation and letting schema-invalid manifests through; meanwhile most manifests carry no `metadata.namespace`, so applying them out of order (only `namespace.yaml` first) risks "namespaces not found" races and cross-environment leakage if `--namespace` is ever forgotten.
   - *Fix:* Keep validation ON (kubeconform already gates bad manifests in `prepare`), drop `--validate=false`, and set explicit `namespace:` fields per overlay instead of relying on a CLI flag.

10. **Debug scaffolding left in the pipeline & cancelled mid-deploy runs**
    - *Problem:* The workflow accumulated noise — a "Debug repo state" step, stray `ls -la ../..`/`pwd` lines inside the kubeconform step, and `concurrency.cancel-in-progress: true`, which can cancel a run *while it is deploying*, leaving a half-applied release.
    - *Fix:* Delete the debug steps once path issues were resolved, and set `cancel-in-progress: false` (or move cancellation to only the `prepare` job) so deploys always finish or fail atomically.

---

## Things to Improve

1. **Adopt Kustomize overlays per environment** — replace fragile `sed` edits with `overlays/staging` and `overlays/production` (namespace, replicas, image tag, configmap diffs). One source of truth, reviewable diffs.
2. **Build & push a real application image** — add a registry (GHCR/ECR) build+scan step in `prepare`, pin images by digest/tag, and set `imagePullPolicy: IfNotPresent`→`Always` appropriately. No more deploying bare `nginx:alpine` as a stand-in.
3. **Add Helm or Argo CD for true GitOps** — let a GitOps controller reconcile clusters from this repo (drift detection, auto-rollback) instead of imperative `kubectl apply` from CI runners holding cluster-admin kubeconfigs.
4. **Real health verification before promotion** — re-enable the commented-out post-production `curl --fail https://example.com/health` check, plus a staging smoke test, and gate production promotion on it (not just human approval).
5. **Automatic rollback strategy** — configure `strategy.rollingUpdate` with sensible maxUnavailable/maxSurge, `progressDeadlineSeconds`, and a CI step running `kubectl rollout undo` when `rollout status` fails.
6. **Security hardening of workloads** — add `securityContext` (runAsNonRoot, readOnlyRootFilesystem, drop capabilities), a NetworkPolicy, and a `PodDisruptionBudget`; validate with a policy engine (Kyverno/OPA/Gatekeeper) and a CVE scanner (Trivy) in CI.
7. **Split secrets management out of Git** — integrate External Secrets Operator, Sealed Secrets, or SOPS; remove `secret.yaml` values entirely and document required secret keys in the README.
8. **CI quality gates** — add YAML lint (`yamllint`), `actionlint`, `kube-score`, and unit/smoke tests against the ephemeral Kind cluster as a dedicated `test` job (that is what Kind *should* be used for), all required checks before deploy jobs run.
9. **Observability & ops readiness** — wire up Prometheus metrics/ServiceMonitor annotations, structured logging config, alerting rules, and document runbooks (how to promote, roll back, rotate the TLS cert referenced by `myapp-tls`).
10. **Repository hygiene & documentation** — delete the orphaned `.github/test` file and the mis-named workflow file, rename `multi1.yaml` to something descriptive (`gitops-deploy.yaml`), template `myapp.example.com`/`staging.example.com` hosts per environment, add proper TLS cert provisioning (cert-manager), and keep this README updated with architecture diagrams and env-var/secret requirements.
