# SQL Server Proxy
Windows worker nodes inside of TKGI provisioned Kubernetes clusters currently do not have out-of-cluster egress access. However, the accompanied Linux worker nodes do. As a work around for accessing external resources, like SQL Server, an HAProxy can be setup as a reverse proxy to access the database.

## HAProxy Config
The below config is a simple example of a reverse proxy to allow applications to access a specific server. Adjust the `10.10.10.10` IP address to point to your server.

``` cfg
global
defaults
  timeout client          30s
  timeout server          30s
  timeout connect         30s

frontend localhost
  bind *:1433
  mode tcp
  use_backend mssql

backend mssql
  mode tcp
  server mssqlserver 10.10.10.10:1433 check

```

## Docker Image
After the config is setup, a Dockerfile with an `haproxy` base image will let you copy the config into a new image.

```docker
FROM haproxy:1.7
COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
EXPOSE 1433

```

You can now run the below command to build the image

```bash
docker build -t haproxy-mssql:1.0.0 ./
```

## Push Docker Image
Login to your container registry
```bash
docker login -u user harbor.registry.domain
```

Tag the image for your registry
```bash
docker tag haproxy-mssql:1.0.0 harbor.registry.domain/repository/haproxy-mssql:1.0.0
```

And push the image
```bash
docker push harbor.registry.domain/repository/haproxy-mssql
```

## Run Deployment
Using the below deployment file
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: haproxy-mssql
spec:
  replicas: 3
  selector:
    matchLabels:
      app: haproxy-mssql
  template:
    metadata:
      labels:
        app: haproxy-mssql
    spec:
      containers:
      - name: haproxy-mssql
        image: harbor.registry.domain/repository/haproxy-mssql:1.0.0
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 1433

```

Apply the deployment to a namespace within your Kubernetes cluster and expose to other pods.
```bash
kubectl create namespace proxy
kubectl -n proxy apply -f ./haproxy-mssql-deployment.yaml
kubectl -n proxy expose deploy haproxy-mssql
```

After the deployment is exposed, a cluster IP address can be used to connect to the database
```bash
kubectl -n proxy get svc
```
