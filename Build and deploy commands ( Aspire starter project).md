
Ref: [Let's start from scratch](https://github.com/devkimchi/aspir8-from-scratch)
Ref: [Aspir8 | Aspir8: Aspire to Deployments ](https://prom3theu5.github.io/aspirational-manifests/getting-started.html)
## Prerequisites

- for Aspire
  - [.NET 8 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/8.0) 8.0.200 or higher with the [Aspire workload](https://learn.microsoft.com/dotnet/aspire/fundamentals/setup-tooling?tabs=dotnet-cli)
  - [Visual Studio Code](https://code.visualstudio.com/) with the [C# Dev Kit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit) extension
- for local Kubernetes cluster
  - [Docker Desktop](https://www.docker.com/products/docker-desktop/)

## Local Kubernetes Cluster Setup through Docker Desktop

1. [Install Docker Desktop on you local machine](https://docs.docker.com/desktop/install/mac-install/).
1. [Enable Kubernetes in Docker Desktop](https://docs.docker.com/desktop/kubernetes/).
1. [Deploy sample app to a Kubernetes cluster](https://docs.docker.com/get-started/kube-deploy/).

## Kubernetes Dashboard Setup

### Use Helm Charts

> **Note:** From Kubernetes Dashboard v3.x, use [Helm Charts](https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard) approach.

1. Install [Helm](https://helm.sh/docs/intro/install/).
2. Run the following commands to install the Kubernetes Dashboard.
```pwsh
# Add kubernetes-dashboard repository
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

# Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

3. Create admin user.
```pwsh
kubectl apply -f ./admin-user.yaml
```

4. Get the access token. Take note the access token to access the dashboard.
```pwsh
kubectl get secret admin-user `
-n kubernetes-dashboard `
-o jsonpath='{ .data.token }' | `
% { [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

5. Run the proxy server.
```pwsh
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```

5. Access the dashboard using the following URL:
```text
https://localhost:8443
```

1. Enter the access token to access the dashboard.
## Aspire-flavoured App Build

1. Install .NET Aspire workload.
```pwsh
dotnet workload update && dotnet workload install aspire
```

2. Create a new Aspire starter app.
```pwsh
dotnet new aspire-starter -n Jintech.NET
```

3. Build the app.
```pwsh
dotnet restore && dotnet build
```

4. Run the app locally.
```pwsh
dotnet run --project Jintech.NET.AppHost
```

5. Open the app in a browser, and go to the weather page to see whether the API is working or not. The port number might be different from the example below.
```text
http://localhost:17008
```

## Aspire-flavoured App Deployment to Kubernetes Cluster through Aspir8

### Use local container registry

1. Install [Distribution (formerly known as Registry)](https://github.com/distribution/distribution) as a local Docker Hub (Container Registry).
```pwsh
docker run -d -p 6000:5000 --name registry registry:latest
```

   > **Note:** The port number of `6000` is just an arbitrary number. You can choose your own one.

2. Install [Aspir8](https://github.com/prom3theu5/aspirational-manifests).
```pwsh
dotnet tool install -g aspirate
```

3. Initialize Aspir8.
```pwsh
cd Aspir8.AppHost
aspirate init -cr localhost:6000 -ct latest --disable-secrets true --non-interactive
```

4. Build and publish the app to the local container registry.
```pwsh
aspirate generate --image-pull-policy Always --namespace jintech --include-dashboard true --disable-secrets true --non-interactive
```

5. Deploy the app to the Kubernetes cluster.
```pwsh
aspirate apply -k docker-desktop --non-interactive
```

6. Check the services in the Kubernetes cluster.
```pwsh
kubectl get services --all-namespaces 
```

7. Install a load balancer for `webfrontend` to the local Kubernetes cluster.
```pwsh
kubectl apply -f ../load-balancer.yaml
```

8. Install a load balancer for `aspire-dashboard` to the local Kubernetes cluster.
```pwsh
kubectl apply -f ../aspire-dashboard.yaml
```

9. Open the app in a browser, and go to the dashboard page to see the logs
```text
http://localhost:18888
```

10. Open the app in a browser, and go to the weather page to see whether the API is working or not.
```text
http://localhost/weather
```

11. Check the services in the Kubernetes cluster.
```pwsh
kubectl get services --all-namespaces 
```

12. Delete all resource from namespace
```pwsh
kubectl delete all --all -n jintech
```
### Balancer

- load-balancer.yaml
```yaml 
apiVersion: v1
kind: Service
metadata:
  name: webfrontend-lb
  namespace: jintech
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: webfrontend
  type: LoadBalancer
```

- aspire-dashboard.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: aspire-dashboard-lb
  namespace: jintech
spec:
  ports:
  - name: http
    port: 18888
    targetPort: 18888
  - name: otlp
    port: 4317
    targetPort: 18889
  selector:
    app: aspire-dashboard
  type: LoadBalancer
```
