## Steps

1. Create Dockerfile => We used the one from the first lab
2. Create a repository on docker hub
3. Push the dockerfile to a registry

```bash
docker build . -t mrfelix/ikt210:latest
docker push mrfelix/ikt210:latest
```ku

4. Create a new namespace for all the resources in this lab

```bash
kubectl create namespace lab02
```

5. Create a deployment file for the application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beetroot-deployment
  namespace: lab02
spec:
  replicas: 1
  selector:
    matchLabels:
      app: beetroot
  template:
    metadata:
      labels:
        app: beetroot
    spec:
      containers:
      - name: beetroot
        image: mrfelix/beetroot:latest
        ports:
        - containerPort: 8080
```

6. Apply deployment file

```bash
kubectl apply -f ./deployment.yaml 
```

7. Create a service file

```yaml
apiVersion: v1
kind: Service
metadata:
  name: beetroot
  namespace: lab02
spec:
  type: NodePort
  selector:
    app: beetroot
  ports:
    - port: 8080
```

8. Check if the service and deployments are running by creating a node-port

```bash
kubectl port-forward service/beetroot 3000:8080 -n lab02
```

9. Open `localhost:3000` in the browser. We now have access to the beetroot application through the port forward.

10. Create liveness probe by adding the following to the container spec in the template section of the deployment manifest:


```yaml
livenessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 3
```

This creates an HTTP Get request to the / route. If it would return any other status code than a 2XX it would fail and restart the application


11. Create deployment with apache proxy

```bash
kubectl apply -f ./apache/deployment.yaml
kubectl apply -f ./apache/service.yaml

kubectl port-forward service/apache 3000:80 -n lab02
```

12. Create deployment with caddy proxy

```bash
kubectl apply -f ./caddy/deployment.yaml
kubectl apply -f ./caddy/service.yaml

kubectl port-forward service/caddy 3000:80 -n lab02
```

13.  Create deployment with nginx proxy

```bash
kubectl apply -f ./nginx/deployment.yaml
kubectl apply -f ./nginx/service.yaml

kubectl port-forward service/nginx 3000:80 -n lab02
```

## Problems

- Image could not be pulled. Pod shows `ImagePullBackOff` state.
  - Repository in Docker Hub was private
- Image shows  `CrashLoopBackOff` state. Error `exec /bin/sh: exec format`
  - Image was build and based on arm64 architecture. In Docker hub it shows the OS as `linux/arm64/v8`. So the node cannot run this version. Solution: Create the docker image on amd64 architecture.
  - Created a repository in github with an action to build and push the image
- port-forward didn't work on the service. Only on the deployment.
  - Issue was the app selector in the service.
- Apache couldn't start because port 8080 was used already by the beetroot application.