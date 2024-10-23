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

### Security Settings
The base deployment has several security configurations to ensure the container is running in a secure environment:

- **Pod Security Context (`securityContext`)**: This section sets security attributes for the entire pod.
  - **`seccompProfile`**: Specifies the type of Seccomp profile. Using `RuntimeDefault` ensures that the default seccomp profile is applied, which restricts the system calls that the container can make, improving security.

- **Container Security Context (`containers[].securityContext`)**: Each container has specific security settings to ensure it runs with minimal privileges.
  - **`allowPrivilegeEscalation: false`**: Prevents the container from gaining more privileges than it started with. This helps mitigate privilege escalation attacks.
  - **`privileged: false`**: Ensures the container is not running in privileged mode, which would allow it to access all devices on the host. This setting significantly reduces the risk of a container breakout.
  - **`runAsNonRoot: true`**: Forces the container to run as a non-root user. Running as a non-root user limits the potential damage if the container is compromised.
  - **`runAsUser: 1000`**: Specifies the user ID (UID) under which the container should run. Using a non-zero UID ensures that the container does not run as the root user.
  - **`capabilities.drop: [ALL]`**: Drops all Linux capabilities for the container. Linux capabilities are granular permissions that a process can have, and dropping all of them minimizes the attack surface.
The **base** directory contains the generic Kubernetes resources that apply to all environments. This includes the main deployment manifest and a `kustomization.yaml` file to aggregate them.

### Base Deployment (`base/deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: web-app
          image: giuffrelab/flask-web-app:v1
          securityContext:
            allowPrivilegeEscalation: false
            privileged: false
            runAsNonRoot: true
            runAsUser: 1000  # Using a non-zero UID to ensure it's not running as root
            capabilities:
              drop:
                - ALL  # Drop all Linux capabilities unless explicitly needed
          env:
            - name: PORT
              value: "8080"  # Keeping the port configuration
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /readiness
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
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
  name: web-app  # Must match the base deployment name for patching
spec:
  replicas: 5  # Override replica count for dev
  template:
    spec:
      containers:
        - name: web-app  # This name is crucial to ensure that Kustomize knows which container to modify
          image: giuffrelab/flask-web-app:v3
          env:
            - name: PORT
              value: "8081"  # Replace the PORT for the dev environment
          ports:
            - containerPort: 8081  # Override containerPort for dev
          readinessProbe:
            httpGet:
              path: /readiness
              port: 8081
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8081
            initialDelaySeconds: 10
            periodSeconds: 15
```

#### Development Service (`overlays/dev/service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service-dev
spec:
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 8081  # External port for development
    targetPort: 8081  # Internal container port, matches the environment variable in the deployment patch
  type: LoadBalancer
```

#### Development Kustomization (`overlays/dev/kustomization.yaml`)
```yaml
resources:
  - ../../base
  - namespace.yaml
  - service.yaml

namespace: dev-web-app

patches:
  - path: patch.yaml
    target:
      kind: Deployment
      name: web-app
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
  name: web-app  # Must match the base deployment name for patching
spec:
  replicas: 5  # Override replica count for prod
  template:
    spec:
      containers:
        - name: web-app  # This name is crucial to ensure that Kustomize knows which container to modify
          image: giuffrelab/flask-web-app:v1
          env:
            - name: PORT
              value: "8082"  # Replace the PORT for the prod environment
          ports:
            - containerPort: 8082  # Override containerPort for prod
          readinessProbe:
            httpGet:
              path: /readiness
              port: 8082
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8082
            initialDelaySeconds: 15
            periodSeconds: 20
```

#### Production Service (`overlays/prod/service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service-prod
spec:
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 8082  # External port for production
    targetPort: 8082  # Internal container port, matches the environment variable in the deployment patch
  type: LoadBalancer
```

#### Production Kustomization (`overlays/prod/kustomization.yaml`)
```yaml
resources:
  - ../../base
  - namespace.yaml
  - service.yaml

namespace: prod-web-app

patches:
  - path: patch.yaml
    target:
      kind: Deployment
      name: web-app
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
     kubectl create -k overlays/dev
     ```
   - To deploy the production environment:
     ```sh
     kubectl create -k overlays/prod
     ```
   - To deploy both environments using the top-level kustomization file:
     ```sh
     kubectl create -k .
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

4. **Accessing the Application**
   - Once the services are running, you can access the application via the LoadBalancer IP or NodePort that was assigned.
   - To get the external IP address of the service, use:
     ```sh
     kubectl get services -n <namespace>
     ```
     Replace `<namespace>` with `dev-web-app` or `prod-web-app`.
   - Open your browser and navigate to the IP address and port of the service. You should see the "Kubernetes Kustomization Test Page" displayed.
   - **Refreshing the Page**: Refresh the page multiple times to see the different containers serving the web view. The hostname displayed on the page will change based on the container serving the request, which helps demonstrate the load balancing across the replicas.

## Rolling Out Updates
Rolling out new versions of the Flask application involves updating the container image version in the Kubernetes manifests and monitoring the rollout process to ensure it completes successfully.

### Step 1: Update the Image Version
To roll out a new version, update the image tag in the deployment manifest for the desired environment. This is helpful for seeing how Kubernetes handles changes of a service to new versions without downtime in a rolling fashion. For example, if you are rolling out version `v2` to the dev environment:

#### Updating the Development Overlay (`overlays/dev/patch.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app  # Must match the base deployment name for patching
spec:
  replicas: 5  # Override replica count for dev
  template:
    spec:
      containers:
        - name: web-app  # This name is crucial to ensure that Kustomize knows which container to modify
          image: giuffrelab/flask-web-app:v3  # Change the version here IE: v1, v2, or v3
          env:
            - name: PORT
              value: "8081"  # Replace the PORT for the dev environment
          ports:
            - containerPort: 8081  # Override containerPort for dev
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
kubectl rollout status deployment/web-app -n <namespace>
```
Replace `<namespace>` with the appropriate namespace (`dev-web-app` or `prod-web-app`). This command will provide information about the status of the deployment and whether the rollout is progressing as expected.

During this time, you should still have access to your web pages, and depending on the update rollout status, you may hit pages that are on different versions of deployment, but your service is never completely down.

### Step 4: Verify the Rollout
Once the rollout is complete, you can verify that the new version is running:

- Get the list of pods to confirm that new pods are running with the updated image:
  ```sh
  kubectl get pods -n <namespace>
  ```
- Describe a specific pod to verify the image version and health checks:
  ```sh
  kubectl describe pod <pod-name> -n <namespace>
  ```
  Check the `Image` field under the container section to confirm the version. You can also view the readiness and liveness probe configurations and their current status to verify that the health checks are working as expected.

### Step 5: Rollback if Necessary
If there are any issues with the new version, you can rollback to the previous version using:
```sh
kubectl rollout undo deployment/web-app -n <namespace>
```
This command will revert the deployment to the previous version.

## Helpful Commands

Here is a list of commands that can be helpful when looking at the full deployment:

### Show All Pods, Services, and Replica Sets for Both Environments
```sh
for ns in dev-web-app prod-web-app; do
  echo "Namespace: $ns"
  echo "Pods:"
  kubectl get pods -n $ns
  echo "Services:"
  kubectl get svc -n $ns
  echo "Replica-Sets:"
  kubectl get rs -n $ns
  echo "-----------------------------"
done
```

### Show Logs from All Pods in a Namespace

#### For Development
```sh
kubectl logs -l app=web-app -n dev-web-app
```

#### For Production
```sh
kubectl logs -l app=web-app -n prod-web-app
```

## Summary
This documentation provides instructions for deploying the Flask application using Kubernetes and Kustomize. The repository structure allows for easy customization of the application for different environments (e.g., development and production) while sharing a common base configuration.

