# Kubernetes Deployment Documentation for Flask Web Application

## Overview
This repository contains Kubernetes configurations for deploying a Flask web application using Kustomize. The repository is organized into base and overlay directories to handle multiple environments (e.g., development and production). The goal of this documentation is to guide you through deploying the application in different environments using Kubernetes.

[GitHub Repo for Flask App Demo](https://github.com/GiuffreLab/flask-app-demo)

## Repository Structure
The repository follows an overlay pattern for Kubernetes manifests:

```
├── base
│   ├── deployment.yaml
│   └── kustomization.yaml
├── kustomization.yaml
└── overlays
    ├── dev
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   ├── patch.yaml
    │   └── service.yaml
    └── prod
        ├── kustomization.yaml
        ├── namespace.yaml
        ├── patch.yaml
        └── service.yaml
```

### Directory Description
- **base**: Contains the base deployment configuration for the Flask application.
- **overlays**: Contains environment-specific configurations, such as development (dev) and production (prod).
  - **dev**: Overlay configuration for the development environment.
  - **prod**: Overlay configuration for the production environment.
- **kustomization.yaml**: The top-level kustomization file to deploy the application for both environments.

## Base Configuration
The **base** directory contains the generic Kubernetes resources that apply to all environments. This includes the main deployment manifest and a `kustomization.yaml` file to aggregate them.

### Base Deployment (`base/deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-web-app
  labels:
    app: flask-web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-web-app
  template:
    metadata:
      labels:
        app: flask-web-app
    spec:
      containers:
      - name: flask-web-app
        image: <your-dockerhub-username>/flask-web-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
```

### Base Kustomization (`base/kustomization.yaml`)
```yaml
resources:
- deployment.yaml
```

## Overlays Configuration
The **overlays** directory contains environment-specific configurations that extend the base configuration.

### Development Overlay (`overlays/dev`)
The **dev** overlay customizes the base configuration for a development environment.

#### Development Namespace (`overlays/dev/namespace.yaml`)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev-web-app
```

#### Development Patch (`overlays/dev/patch.yaml`)
This patch modifies the base deployment to suit the development environment.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-web-app
spec:
  replicas: 3  # Override replica count for dev
  template:
    spec:
      containers:
      - name: flask-web-app
        env:
        - name: PORT
          value: "8081"  # Replace the PORT for the dev environment
```

#### Development Service (`overlays/dev/service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-web-app-service-dev
  namespace: dev-web-app
spec:
  selector:
    app: flask-web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8081
  type: LoadBalancer
```

#### Development Kustomization (`overlays/dev/kustomization.yaml`)
```yaml
resources:
  - ../../base
  - namespace.yaml
  - service.yaml

patchesStrategicMerge:
  - patch.yaml
```

### Production Overlay (`overlays/prod`)
The **prod** overlay customizes the base configuration for a production environment.

#### Production Namespace (`overlays/prod/namespace.yaml`)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod-web-app
```

#### Production Patch (`overlays/prod/patch.yaml`)
This patch modifies the base deployment to suit the production environment.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-web-app
spec:
  replicas: 5  # Override replica count for production
  template:
    spec:
      containers:
      - name: flask-web-app
        env:
        - name: PORT
          value: "8082"  # Replace the PORT for the production environment
```

#### Production Service (`overlays/prod/service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-web-app-service-prod
  namespace: prod-web-app
spec:
  selector:
    app: flask-web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8082
  type: LoadBalancer
```

#### Production Kustomization (`overlays/prod/kustomization.yaml`)
```yaml
resources:
  - ../../base
  - namespace.yaml
  - service.yaml

patchesStrategicMerge:
  - patch.yaml
```

## Top-Level Kustomization
To deploy the application for both environments, use the top-level `kustomization.yaml` file.

### Top-Level Kustomization (`kustomization.yaml`)
```yaml
resources:
  - overlays/dev
  - overlays/prod
```

## Viewing the Kustomize Structure
To view how the overlays modify the base configuration, you can use the `kubectl kustomize` command to generate the manifests before applying them. This is helpful to verify the changes made by each overlay.

### Viewing the Base Configuration
To see the base deployment configuration, run:
```sh
kubectl kustomize base
```
This command will output the manifests defined in the `base` directory, including the deployment with the default settings.

### Viewing the Development Overlay
To see how the development overlay modifies the base configuration, run:
```sh
kubectl kustomize overlays/dev
```
This command will output the manifests with the development-specific changes applied, such as the increased replica count and updated port configuration.

### Viewing the Production Overlay
To see how the production overlay modifies the base configuration, run:
```sh
kubectl kustomize overlays/prod
```
This command will output the manifests with the production-specific changes, including the increased replica count and different port settings.

By using these commands, you can verify that the overlays are correctly modifying the base configuration to meet the needs of each environment.

## Deploying the Application
1. **Select the Environment**
   - You can use `kubectl kustomize` to generate the Kubernetes manifests for a specific environment (e.g., dev or prod).

2. **Deploy the Application**
   - To deploy the development environment:
     ```sh
     kubectl apply -k overlays/dev
     ```
   - To deploy the production environment:
     ```sh
     kubectl apply -k overlays/prod
     ```
   - To deploy both environments using the top-level kustomization file:
     ```sh
     kubectl apply -k .
     ```

3. **Verify the Deployment**
   - Check the status of the deployments and services:
     ```sh
     kubectl get deployments -n dev-web-app
     kubectl get services -n dev-web-app
     ```
     ```sh
     kubectl get deployments -n prod-web-app
     kubectl get services -n prod-web-app
     ```

## Rolling Out New Versions
Rolling out new versions of the Flask application involves updating the container image version in the Kubernetes manifests and monitoring the rollout process to ensure it completes successfully.

### Step 1: Update the Image Version
To roll out a new version, update the image tag in the deployment manifest for the desired environment. For example, if you are rolling out version `v2`:

#### Updating the Development Overlay (`overlays/dev/patch.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-web-app
spec:
  replicas: 3  # Override replica count for dev
  template:
    spec:
      containers:
      - name: flask-web-app
        image: <your-dockerhub-username>/flask-web-app:v2  # Update image version
        env:
        - name: PORT
          value: "8081"  # Replace the PORT for the dev environment
```

### Step 2: Apply the Changes
After updating the image version, apply the changes to the cluster:

- For the development environment:
  ```sh
  kubectl apply -k overlays/dev
  ```
- For the production environment:
  ```sh
  kubectl apply -k overlays/prod
  ```

### Step 3: Monitor the Rollout
To monitor the progress of the rollout, use the following command:
```sh
kubectl rollout status deployment/flask-web-app -n <namespace>
```
Replace `<namespace>` with the appropriate namespace (`dev-web-app` or `prod-web-app`). This command will provide information about the status of the deployment and whether the rollout is progressing as expected.

### Step 4: Verify the Rollout
Once the rollout is complete, you can verify that the new version is running:

- Get the list of pods to confirm that new pods are running with the updated image:
  ```sh
  kubectl get pods -n <namespace>
  ```
- Describe a specific pod to verify the image version:
  ```sh
  kubectl describe pod <pod-name> -n <namespace>
  ```
  Check the `Image` field under the container section to confirm the version.

### Step 5: Rollback if Necessary
If there are any issues with the new version, you can rollback to the previous version using:
```sh
kubectl rollout undo deployment/flask-web-app -n <namespace>
```
This command will revert the deployment to the previous version.

## Summary
This documentation provides instructions for deploying the Flask application using Kubernetes and Kustomize. The repository structure allows for easy customization of the application for different environments (e.g., development and production) while sharing a common base configuration.

