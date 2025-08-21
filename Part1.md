# Helm Assignment Guide

This guide documents the conceptual answers and practical examples for
the **Helm Home Assignment**.\
It covers Helm basics, environment-specific configurations,
repositories, and CI/CD integration.

------------------------------------------------------------------------

## 1. Helm's Role in Kubernetes

### What is Helm?

Helm is the **package manager for Kubernetes**, allowing you to bundle
related manifests into a **chart**.\
You can install, upgrade, share, and roll back applications as a single
unit.

### Why Helm instead of plain YAML?

-   **Templating & DRY**: Write once, reuse with `values.yaml`.
-   **Release management**: Versioned, upgradeable, and rollback
    capable.
-   **Atomic upgrades**: Automatic rollback on failure with `--atomic`.
-   **Distribution**: Charts are packaged and shareable.
-   **Hooks & tests**: Pre/post actions and built-in tests.

### Helm Chart Components

    mychart/
    ├── Chart.yaml        # Metadata about the chart
    ├── values.yaml       # Default configuration values
    ├── templates/        # Templated Kubernetes manifests
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── _helpers.tpl
    └── charts/           # Dependencies (sub charts)

**Chart.yaml**

``` yaml
apiVersion: v2
name: mychart
description: Demo web app
type: application
version: 0.1.0
appVersion: "1.0.0"
```

**values.yaml**

``` yaml
image:
  repository: nginx
  tag: "1.25-alpine"
  pullPolicy: IfNotPresent

replicaCount: 2

service:
  type: ClusterIP
  port: 80
```

**templates/deployment.yaml**

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "mychart.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "mychart.name" . }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 80
```

**templates/service.yaml**

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ include "mychart.name" . }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
```

------------------------------------------------------------------------

## 2. Environment-Specific Configurations

Helm uses **values files** for environment overrides.

**values-dev.yaml**

``` yaml
replicaCount: 1
image:
  tag: "1.25-alpine-dev"
service:
  type: NodePort
env:
  LOG_LEVEL: debug
```

**values-prod.yaml**

``` yaml
replicaCount: 4
image:
  tag: "1.25-alpine"
service:
  type: LoadBalancer
env:
  LOG_LEVEL: warn
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

### Commands

``` bash
helm template myapp ./mychart -f values-dev.yaml
helm upgrade --install myapp-dev ./mychart -n dev -f values-dev.yaml --wait --atomic
helm upgrade --install myapp-prod ./mychart -n prod -f values-prod.yaml --wait --atomic
```

------------------------------------------------------------------------

## 3. Helm Chart Repositories

A **Helm chart repository** is a place to host packaged charts and an
`index.yaml`.\
Charts can be hosted as HTTP repos or OCI artifacts.

### Options

1.  **GitHub Pages**

    ``` bash
    helm package ./mychart
    helm repo index . --url https://<user>.github.io/helm-charts
    ```

2.  **ChartMuseum (self-hosted)**

    ``` bash
    helm repo add mycharts http://chartmuseum.local:8080
    helm cm-push mychart-0.1.0.tgz mycharts
    ```

3.  **Harbor (OCI)**

    ``` bash
    helm push mychart-0.1.0.tgz oci://harbor.local/library
    ```

4.  **Docker Hub / GHCR (OCI)**

    ``` bash
    helm push mychart-0.1.0.tgz oci://ghcr.io/<owner>/<repo>
    ```

5.  **JFrog Artifactory** (HTTP/OCI)

------------------------------------------------------------------------

## 4. CI/CD Integration

Typical pipeline:

1.  **Lint chart**\
    `helm lint mychart`
2.  **Render manifests**\
    `helm template myapp ./mychart -f values-dev.yaml`
3.  **Validate manifests** (e.g., kubeconform)
4.  **Package & push chart**\
    `helm package mychart && helm push ...`
5.  **Deploy with Helm**\
    `helm upgrade --install ...`
6.  **Run Helm tests**\
    `helm test myapp`

### GitHub Actions Example

``` yaml
name: CI-CD Helm

on:
  push:
    branches: [ "main" ]

jobs:
  build-validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
      - run: helm lint mychart
      - run: helm template myapp ./mychart -f mychart/values-dev.yaml > rendered.yaml

  deploy-dev:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
      - run: helm upgrade --install myapp-dev ./mychart -n dev -f mychart/values-dev.yaml --wait --atomic
```

------------------------------------------------------------------------

## Useful Helm Commands

``` bash
helm create mychart
helm lint mychart
helm template myapp ./mychart -f values-dev.yaml
helm upgrade --install myapp ./mychart -f values-dev.yaml --wait --atomic
helm history myapp
helm rollback myapp 1
helm package ./mychart
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo bitnami/redis
```

------------------------------------------------------------------------

✅ With this README.md you can learn, practice, and present your Helm
assignment clearly.
