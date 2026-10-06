# K3s Container Image Versioning with ArgoCD and GitOps

Implementing GitOps-based application deployment on Kubernetes K3s using ArgoCD, Helm, GitLab CI/CD, and GitLab Container Registry.

## Overview

This project implements the third deployment method using **ArgoCD and GitOps** on a Kubernetes K3s cluster.

Unlike the previous K3s deployment method using `kubectl`, GitLab CI/CD does not directly deploy the application to the Kubernetes cluster.

The responsibilities are separated:

* **GitLab CI/CD** builds the Docker image.
* **GitLab Container Registry** stores the Docker image.
* **Helm** stores the Kubernetes deployment configuration.
* **Git repository** stores the desired state.
* **ArgoCD** monitors the Git repository and synchronizes the desired state to K3s.
* **Kubernetes K3s** runs the resulting application resources.

The Git repository therefore becomes the **source of truth** for the application deployment configuration.

---

## GitOps Concept

GitOps is a deployment approach where Git is used as the primary source of truth for application and infrastructure configuration.

The application source code is stored in GitLab. When a new application version is released, a Git Tag is created.

GitLab CI/CD then builds a Docker image using the Git Tag as the image version.

For example:

```text
Git Tag:
v0.1.19
```

The resulting image can be:

```text
registry.gitlab.com/k3s/argocd/franken-php:v0.1.19
```

The image is pushed to GitLab Container Registry.

The image version used by Kubernetes is defined separately in the Helm repository. ArgoCD monitors this repository and compares the desired state stored in Git with the actual state of the K3s cluster.

When the Helm configuration changes, ArgoCD detects the difference and synchronizes the application according to the configured synchronization policy.

The basic relationship is:

```text
Source Code
     |
     v
GitLab CI/CD
     |
     v
Docker Image
     |
     v
GitLab Container Registry

Helm Repository
     |
     v
ArgoCD
     |
     v
Kubernetes K3s
```

The important point is that ArgoCD does not automatically select the newest image from the registry. The image version must be defined in the Git-based deployment configuration.

---

## Objectives

This implementation has several objectives:

* Build Docker images using GitLab CI/CD.
* Use Git Tags for image versioning.
* Store images in GitLab Container Registry.
* Store Kubernetes configuration in a Git repository.
* Manage Kubernetes configuration using Helm.
* Use ArgoCD as the GitOps controller.
* Automatically synchronize application changes to K3s.
* Keep deployment configuration version-controlled.
* Provide traceability through Git commits and image versions.
* Separate image building from application deployment.

---

## Architecture

The deployment architecture consists of several components.

### GitLab CI/CD

GitLab CI/CD is responsible for building and pushing Docker images.

It does not directly deploy the application to K3s.

### GitLab Container Registry

The registry stores the Docker images produced by the CI/CD pipeline.

Example:

```text
registry.gitlab.com/k3s/argocd/franken-php:v0.1.19
```

### Helm Repository

The Helm repository contains the desired Kubernetes configuration.

The image repository and image tag are stored in `values.yaml`.

### ArgoCD

ArgoCD monitors the Helm repository and compares the desired state with the current state of the Kubernetes cluster.

When a change is detected, ArgoCD can synchronize the application automatically.

### K3s

K3s is the Kubernetes cluster where the application workloads are executed.

---

## K3s Environment

The example K3s environment uses:

| Component        | Example             |
| ---------------- | ------------------- |
| Hostname         | `k3s-vm`            |
| Operating System | Debian GNU/Linux 13 |
| IP Address       | `192.168.1.100`     |
| K3s Version      | `v1.36.4+k3s1`      |
| Namespace        | `argocd`            |

The IP address above is only an example.

Basic environment information can be checked using:

```bash id="k3senv01"
hostname
hostname -I
kubectl version
```

The K3s node can be checked using:

```bash id="k3senv02"
sudo kubectl get nodes
```

The node should be in the `Ready` state.

---

## Install and Verify ArgoCD

ArgoCD runs inside the K3s cluster.

The ArgoCD namespace can be checked using:

```bash id="argocd01"
sudo kubectl get pods -n argocd
```

Typical ArgoCD components include:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-applicationset-controller
argocd-dex-server
argocd-redis
```

The Pods should be in a running state before creating an ArgoCD Application.

---

## ArgoCD Web Interface

ArgoCD provides a web interface for monitoring applications.

An example public hostname is:

```text
https://argocd.example.com
```

The hostname should be replaced with the actual domain configured for the environment.

The ArgoCD interface can be used to inspect:

* Application status
* Sync status
* Health status
* Git repository
* Git revision
* Helm configuration
* Kubernetes resources
* Deployment
* Service
* Pod

This provides a centralized view of the application without having to inspect every Kubernetes resource manually.

---

## ArgoCD Application

The ArgoCD Application used for this implementation is:

```text
image-versioning-argocd
```

The Application points to a Git repository containing the Helm Chart.

Example repository:

```text
git@gitlab.com:example/argocd/helm.git
```

The Helm Chart path is:

```text
chart
```

The destination namespace is:

```text
default
```

The Application configuration defines:

* Git repository
* Helm Chart path
* Kubernetes cluster
* Target namespace
* Synchronization policy

The Git repository acts as the desired state for the application.

---

## Synchronization Policy

The ArgoCD Application can use the following synchronization settings:

* Automated Sync
* Prune
* Self Heal

### Automated Sync

Automated Sync allows ArgoCD to synchronize the application automatically when the desired state in Git changes.

### Prune

Prune allows ArgoCD to remove resources that are no longer defined in the Git configuration.

### Self Heal

Self Heal allows ArgoCD to restore resources when the actual cluster state is changed manually and becomes different from the desired state stored in Git.

Together, these settings help keep the Kubernetes cluster aligned with the Git repository.

---

## Helm Chart Structure

The deployment configuration is stored as a Helm Chart.

The repository structure is:

```text
helm/
└── chart/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── deployment.yaml
        └── service.yaml
```

### Chart.yaml

`Chart.yaml` contains the basic metadata of the Helm Chart, including the chart name and version.

### values.yaml

`values.yaml` contains configurable application values.

The Docker image repository and version are defined here.

### templates

The `templates` directory contains Kubernetes resource templates.

For this implementation, the main resources are:

* Deployment
* Service

---

## Image Configuration

The image configuration is stored in `values.yaml`.

Example:

```yaml id="imgcfg01"
image:
  repository: registry.gitlab.com/k3s/argocd/franken-php
  tag: v0.1.19
```

The resulting image is:

```text
registry.gitlab.com/k3s/argocd/franken-php:v0.1.19
```

When a new version is available, the tag can be updated.

For example:

```yaml id="imgcfg02"
image:
  repository: registry.gitlab.com/k3s/argocd/franken-php
  tag: v0.1.20
```

This change represents a new desired state for the application.

The new configuration must be committed and pushed to Git before ArgoCD can synchronize it.

---

## GitLab CI/CD Build Process

GitLab CI/CD remains responsible for building the Docker image.

Before building and pushing the image, the pipeline authenticates against GitLab Container Registry:

```bash id="cicd01"
echo "${CI_REGISTRY_PASSWORD}" | docker login "${CI_REGISTRY}" \
  -u "${CI_REGISTRY_USER}" \
  --password-stdin
```

The image is built using the Git Tag:

```bash id="cicd02"
docker build \
  -t "${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}" .
```

The image is then pushed to the registry:

```bash id="cicd03"
docker push "${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}"
```

Using `CI_COMMIT_TAG` ensures that the Docker image version follows the Git release version.

For example:

```text
Git Tag:
v0.1.19

Image:
registry.gitlab.com/k3s/argocd/franken-php:v0.1.19
```

---

## Registry Result

After the Docker push succeeds, the image is available in GitLab Container Registry.

Example:

```text
registry.gitlab.com/k3s/argocd/franken-php:v0.1.19
```

The registry is responsible for storing and distributing the image.

It does not determine when the application should be deployed.

The desired image version is controlled by the Helm configuration stored in Git.

---

## Update Helm Configuration

After a new image has been pushed to the registry, the Helm configuration can be updated.

For example, the previous configuration:

```yaml id="helmup01"
image:
  repository: registry.gitlab.com/k3s/argocd/franken-php
  tag: v0.1.19
```

can be changed to:

```yaml id="helmup02"
image:
  repository: registry.gitlab.com/k3s/argocd/franken-php
  tag: v0.1.20
```

Check the Git changes:

```bash id="helmup03"
git status
```

Add the changes:

```bash id="helmup04"
git add .
```

Create a commit:

```bash id="helmup05"
git commit -m "Update FrankenPHP image to v0.1.20"
```

Push the change:

```bash id="helmup06"
git push origin main
```

The Git push is an important part of the GitOps workflow because it updates the desired state monitored by ArgoCD.

---

## ArgoCD Detects the Change

After the Helm repository is updated, ArgoCD refreshes the repository and compares the desired state with the actual cluster state.

For example, the cluster may currently use:

```text
v0.1.19
```

while Git now defines:

```text
v0.1.20
```

ArgoCD detects this difference.

The difference represents a change in the desired state.

If Automated Sync is enabled, ArgoCD synchronizes the application automatically.

---

## Automated Deployment

With Automated Sync enabled, GitLab CI/CD does not need to execute:

```bash
kubectl apply
```

for the application deployment.

Instead, the process becomes:

```text
Git Tag
   |
   v
GitLab CI/CD
   |
   v
Build Docker Image
   |
   v
GitLab Container Registry
   |
   |
   +--------------------+
                        |
                        v
                  Update Helm
                        |
                        v
                    Git Push
                        |
                        v
                      ArgoCD
                        |
                        v
                 Kubernetes K3s
```

GitLab CI/CD focuses on building the image, while ArgoCD controls the deployment based on the Git repository.

---

## Registry Authentication in Kubernetes

Even though ArgoCD controls the deployment, Kubernetes still requires credentials when pulling a private Docker image.

The Kubernetes Secret used in this implementation is:

```text
gitlab-registry
```

Check the Secret:

```bash id="secret01"
sudo kubectl -n default get secret gitlab-registry
```

Check its type:

```bash id="secret02"
sudo kubectl -n default get secret gitlab-registry \
  -o jsonpath="{.type}"
```

Expected type:

```text
kubernetes.io/dockerconfigjson
```

The Secret is referenced by the application through `imagePullSecrets`.

This allows Kubernetes to authenticate against GitLab Container Registry when pulling the private image.

---

## Deployment Through Helm

When `values.yaml` contains:

```yaml id="deploy01"
image:
  repository: registry.gitlab.com/k3s/argocd/franken-php
  tag: v0.1.20
```

the Helm Chart generates the Kubernetes resources using that image version.

The resulting image is:

```text
registry.gitlab.com/k3s/argocd/franken-php:v0.1.20
```

ArgoCD applies the desired state to the K3s cluster.

Kubernetes then performs the required Deployment update and starts Pods using the new image.

---

## Verify ArgoCD Application

The ArgoCD Application can be checked using:

```bash id="verify01"
sudo kubectl get applications -n argocd
```

The Application name is:

```text
image-versioning-argocd
```

A successful deployment should result in:

```text
SYNC STATUS : Synced
HEALTH      : Healthy
```

### Synced

`Synced` means the application state in Kubernetes matches the desired state defined in Git.

### Healthy

`Healthy` indicates that the resources managed by ArgoCD are operating normally according to their health status.

---

## Verify Deployments

Kubernetes Deployments can be inspected using:

```bash id="verify02"
sudo kubectl get deployments -n default
```

This verifies that the Deployment generated by the Helm Chart exists and has the expected replica count.

---

## Verify Pods

Application Pods can be checked using:

```bash id="verify03"
sudo kubectl get pods -n default -o wide
```

The application Pod should normally be in:

```text
Running
```

The `-o wide` option also provides information such as the Pod IP and Kubernetes node.

---

## Verify Services

Services can be checked using:

```bash id="verify04"
sudo kubectl get services -n default
```

Services provide stable network endpoints for application communication.

---

## Verify the Running Image

The image used by the application can be inspected using:

```bash id="verify05"
sudo kubectl get pods \
  -n default \
  -o jsonpath="{.items[*].spec.containers[*].image}"
```

Example:

```text
registry.gitlab.com/k3s/argocd/franken-php:v0.1.20
```

This confirms that the running workload uses the image version defined by the Helm configuration.

---

## Image Update Workflow

A typical image update looks like this.

### Previous Version

```text
v0.1.19
```

The Helm configuration contains:

```yaml id="workflow01"
image:
  repository: registry.gitlab.com/k3s/argocd/franken-php
  tag: v0.1.19
```

### New Version

A new source code release is created:

```text
v0.1.20
```

GitLab CI/CD builds:

```text
registry.gitlab.com/k3s/argocd/franken-php:v0.1.20
```

and pushes it to GitLab Container Registry.

The Helm configuration is then updated:

```yaml id="workflow02"
image:
  repository: registry.gitlab.com/k3s/argocd/franken-php
  tag: v0.1.20
```

The change is committed:

```bash id="workflow03"
git add .
git commit -m "Update FrankenPHP image to v0.1.20"
git push origin main
```

ArgoCD detects the Git change and synchronizes the application.

Kubernetes then performs the Deployment update and starts the new Pod using:

```text
registry.gitlab.com/k3s/argocd/franken-php:v0.1.20
```

---

## Important GitOps Principle

ArgoCD does not automatically deploy the newest image available in the registry.

For example, if the registry contains:

```text
v0.1.19
v0.1.20
v0.1.21
```

but `values.yaml` contains:

```yaml id="gitops01"
image:
  tag: v0.1.19
```

ArgoCD will continue to use `v0.1.19`.

To deploy `v0.1.21`, the desired state must be updated:

```yaml id="gitops02"
image:
  tag: v0.1.21
```

The change must then be committed and pushed to Git.

This is an important characteristic of GitOps: **the Git repository determines the desired deployment state.**

---

## GitOps Verification

After synchronization, the application can be verified at several levels.

### ArgoCD

```bash id="gitops03"
sudo kubectl get applications -n argocd
```

Expected state:

```text
Synced
Healthy
```

### Kubernetes Deployment

```bash id="gitops04"
sudo kubectl get deployments -n default
```

### Kubernetes Pods

```bash id="gitops05"
sudo kubectl get pods -n default
```

### Kubernetes Services

```bash id="gitops06"
sudo kubectl get services -n default
```

### Running Image

```bash id="gitops07"
sudo kubectl get pods \
  -n default \
  -o jsonpath="{.items[*].spec.containers[*].image}"
```

These checks verify both the ArgoCD synchronization state and the actual Kubernetes workload.

---

## Security Considerations

This repository is intended to be public.

Do not commit:

* Registry passwords
* GitLab access tokens
* SSH private keys
* Kubernetes credentials
* Production `.env` files
* Real private infrastructure credentials
* Real registry authentication Secrets

Use placeholders in public documentation.

The `gitlab-registry` Secret should be created securely in the target Kubernetes cluster or through an appropriate secret-management mechanism.

---

## Result

The ArgoCD implementation successfully applies the GitOps deployment model to Kubernetes K3s.

The workflow separates image creation from application deployment:

1. Developer updates application source code.
2. A Git Tag is created.
3. GitLab CI/CD builds the Docker image.
4. The image is tagged using `CI_COMMIT_TAG`.
5. The image is pushed to GitLab Container Registry.
6. The Helm repository is updated with the desired image version.
7. The change is committed and pushed to Git.
8. ArgoCD detects the Git change.
9. ArgoCD synchronizes the desired state.
10. Kubernetes updates the application.
11. ArgoCD and Kubernetes resources are verified.

This provides a traceable deployment process where application configuration is maintained in Git.

---

## Conclusion

This project demonstrates **container image versioning with Kubernetes K3s using ArgoCD and GitOps**.

GitLab CI/CD is responsible for building and pushing Docker images, while the Helm repository stores the desired deployment configuration.

ArgoCD monitors the Helm repository and synchronizes changes to the K3s cluster. With Automated Sync, application deployment no longer needs to be performed directly by the CI/CD pipeline using `kubectl apply`.

The image version is controlled through the Helm `values.yaml` configuration. When the image version changes, the change is committed to Git and becomes the new desired state.

This approach provides clear separation between:

* **Build** → GitLab CI/CD
* **Image storage** → GitLab Container Registry
* **Deployment configuration** → Git + Helm
* **GitOps synchronization** → ArgoCD
* **Container orchestration** → Kubernetes K3s

The result is a more declarative and traceable deployment workflow where Git becomes the source of truth for the application's desired state.
