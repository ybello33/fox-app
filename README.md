# Fox Autoscaler Project 🦊

This project implements a Kubernetes-based application deployment for a Counter Service and a Fox App. The deployment is enhanced with Kubernetes Event-Driven Autoscaling (KEDA) to dynamically scale the Fox App based on the Counter Service’s metrics.

### Features
1. Counter Service: A Python app that provides metrics for scaling via its REST API.
2. Fox App: A simple Nginx application that scales dynamically.
3. Autoscaling: Powered by KEDA and triggered using a Metric API Scaler.
4. Helm Chart: Simplified deployment using Helm for templating Kubernetes resources.

### Prerequisites
1. Kubernetes Cluster: A running Kubernetes environment on DockerHub Desktop --> https://www.docker.com/products/docker-desktop/
3. Helm: Installed --> https://formulae.brew.sh/formula/helm
4. KEDA: Installed in the Kubernetes cluster
   ```bash
   kubectl apply -f https://github.com/kedacore/keda/releases/download/v2.14.0/keda-operator.v2.14.0.yaml
   ````
### Implementation
Explain how to install or use the project locally, Include code snippets.

#### Step 1: Clone the repository:
```bash
 git clone https://github.com/ybello33/fox-app
```
#### Step 2: Build & Push the "counter-service"
```bash
# Use a Python base image
FROM python:3.8-slim

# Set the working directory
WORKDIR /app

# Copy the source code to the container
COPY src/ /app/

# Install dependencies
RUN pip install -r requirements.txt

# Expose the port the app will run on
EXPOSE 8000

# Command to run the application
CMD ["python", "main.py"]
```
1. Build the Docker Image for the Counter Service locally.
```bash
docker build -t ybello/counter-service:latest .
```
2. Pushing the Docker Image for the Counter Service into Docker Hub -->    https://hub.docker.com/repositories/ybello
```bash
docker push ybello/counter-service:latest
```
#### Step 3: Counter Service Kubernetes Manifests 
##### 1. Create a Deployment for the Counter Service. In the "default" namespace
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: counter-service
  namespace: default
  labels:
    app: counter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: counter
  template:
    metadata:
      labels:
        app: counter
    spec:
      containers:
        - name: counter-service
          image: ybello/counter-service:latest
          ports:
            - containerPort: 8000
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /
              port: 8000
            initialDelaySeconds: 3
            periodSeconds: 5
```
##### 2. Service for Counter Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: counter-service
  labels:
    app: counter
spec:
  selector:
    app: counter
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: ClusterIP
```
#### Step 4: Fox App Kubernetes Manifests
##### 1. Deployment for Fox App
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fox-app
  labels:
    app: fox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fox
  template:
    metadata:
      labels:
        app: fox
    spec:
      containers:
        - name: fox-app
          image: nginx
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
```
##### 2. Service for Fox App
```yaml
apiVersion: v1
kind: Service
metadata:
  name: fox-service
  labels:
    app: fox
spec:
  selector:
    app: fox
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```
##### 3. Apply the Manifests
```bash
kubectl apply -f counter-deployment.yaml
kubectl apply -f counter-service.yaml
kubectl apply -f fox-deployment.yaml
kubectl apply -f fox-service.yaml
```
##### 4. Verify the deployments and services
```bash
kubectl get deployments
kubectl get services
kubectl get pods
```
#### Step 5: Keda for autoscaling the Fox app based on the count metric from the Counter Service.
##### 1. verify that the KEDA components are installed and running in the keda namespace
```bash
kubectl get pods -n keda
```
![Screenshot 2025-01-11 at 16 25 24](https://github.com/user-attachments/assets/2e976ce4-c9e6-43de-8f74-97b7119e45f9)

##### 2. Create a ScaledObject to use KEDA’s Metric API scaler for scaling the Fox app based on the count field of the Counter Service.
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: fox-scaler
  namespace: default
  labels:
    app: fox
spec:
  scaleTargetRef:
    name: fox-app
  pollingInterval: 10  # Check metric every 10 seconds
  cooldownPeriod: 30   # Scale down after 30 seconds of inactivity
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
    - type: metrics-api
      metadata:
        scalerAddress: http://counter-service.default.svc.cluster.local:80/
        metricName: foxes
        value: "5"  # Scale up when `count` exceeds 5
```
##### 3. Apply the ScaledObject
```bash
kubectl apply -f fox-scaler.yaml
```
##### 3. Verify that the ScaledObject has been created and is active
```bash
kubectl get scaledobject
```
![Screenshot 2025-01-11 at 16 23 18](https://github.com/user-attachments/assets/cf099ab5-b377-4d0d-9b32-836fee05c5dc)








 




