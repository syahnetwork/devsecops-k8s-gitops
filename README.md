# ghactions — GitOps-Driven Multi-Environment Deployment

[![GitOps Multi-Environment Deploy](https://github.com/syahnetwork/ghactions/actions/workflows/deploy.yaml/badge.svg)](https://github.com/syahnetwork/ghactions/actions/workflows/deploy.yaml)

A fully automated, GitOps-driven CI/CD pipeline that deploys a containerized application to **staging** and **production** Kubernetes environments. Every change is validated, tagged, deployed to staging automatically, and promoted to production behind a manual approval gate.

---

## Overview

This repository demonstrates a production-ready GitHub Actions workflow for deploying to multiple Kubernetes environments with the following guarantees:

- **Traceable versions** — every deployment is tagged with the short commit SHA, so the exact code running in any environment is queryable from Git history.
- **Progressive promotion** — staging deploys automatically on merge to `main`; production requires staging to succeed *and* a human approval.
- **Environment isolation** — `staging` and `production` use separate GitHub Environments with separate secrets, so credentials never leak across boundaries.
- **Fail-fast validation** — Kubernetes manifests are validated with `kubeconform` before any cluster is touched.
- **Rollout verification** — the pipeline waits for `kubectl rollout status`, so a green check means the Pods are actually `Ready`.


