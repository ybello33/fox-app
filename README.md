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
![Screenshot 2025-01-11 at 16 27 30](https://github.com/user-attachments/assets/e834fd9f-3f23-4d2b-bb87-23c0ccc125ef)

#### Step 5: Keda for autoscaling the Fox app based on the count metric from the Counter Service.
##### 1. verify that the KEDA components are installed and running in the keda namespace
```bash
kubectl get pods -n keda
```
![Screenshot 2025-01-11 at 16 25 24](https://github.com/user-attachments/assets/2e976ce4-c9e6-43de-8f74-97b7119e45f9)

##### 2. Create a ScaledObject to use KEDA’s Metric API scaler for scaling the Fox app based on the count field of the Counter Service.
fox-scaler.yaml
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

#### Step 5: Testing Autoscaling
##### 1. Scale Up Increase the Counter Service value using the /plusone endpoint
```bash
curl http://local-ip/plusone
```
![Screenshot 2025-01-11 at 16 52 30](https://github.com/user-attachments/assets/ebbb3ecf-62a5-473d-9c40-821bd2a20b12)



#### Option of Using Helm
##### 1. Create a Helm Chart
```bash
helm create fox-autoscaler
```
```helm
This will generate the basic Helm chart structure:
fox-autoscaler/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── scaler.yaml
│   └── _helpers.tpl
```
##### 2. Templates Definition in the Chart
deployment.yaml: (Counter and Fox Apps)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.counter.name }}
  labels:
    app: counter
spec:
  replicas: {{ .Values.counter.replicas }}
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
          image: {{ .Values.counter.image }}
          ports:
            - containerPort: {{ .Values.counter.port }}
          readinessProbe:
            httpGet:
              path: /
              port: {{ .Values.counter.port }}
            initialDelaySeconds: 3
            periodSeconds: 5
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fox-app
  labels:
    app: fox
spec:
  replicas: {{ .Values.fox.replicas }}
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
          image: {{ .Values.fox.image }}
          ports:
            - containerPort: {{ .Values.fox.port }}
```
service.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: counter-service
spec:
  selector:
    app: counter
  ports:
    - protocol: TCP
      port: 80
      targetPort: {{ .Values.counter.port }}
---
apiVersion: v1
kind: Service
metadata:
  name: fox-service
spec:
  selector:
    app: fox
  ports:
    - protocol: TCP
      port: 80
      targetPort: {{ .Values.fox.port }}
```
scaler.yaml:
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
  pollingInterval: 10
  cooldownPeriod: 30
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
    - type: metrics-api
      metadata:
        scalerAddress: {{ .Values.counter.metricEndpoint }}
        metricName: foxes
        value: "{{ .Values.scaling.threshold }}"
```
Define values.yaml
```yaml
counter:
  name: counter-service
  image: ybello/counter-service:latest
  replicas: 1
  port: 8000
  metricEndpoint: http://counter-service.default.svc.cluster.local:80/

fox:
  name: fox-app
  image: nginx
  replicas: 1
  port: 80

scaling:
  threshold: 5
```
##### 3. Deploy the Chart
```bash
helm install fox-autoscaler ./fox-autoscaler
```
Verify deployment:
```bash
helm list
kubectl get pods
```

### Docker Desktop - Info 
#### 1. Deployment Services
![Screenshot 2025-01-11 at 17 25 02](https://github.com/user-attachments/assets/f0ec329a-2592-49e5-adfb-5e40302b62cd)

#### 2. Services
![Screenshot 2025-01-11 at 17 25 24](https://github.com/user-attachments/assets/8fa1a26d-8edc-462c-aff4-9645912ad1fa)

#### 3. Pods per Services & namespacs


#### 4. Endpoints


#### 5. Helm Charts

#### 6. Helm Release




 




