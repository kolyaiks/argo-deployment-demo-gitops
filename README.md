## argo-deployment-demo-gitops

This repository serves as the GitOps source of truth for ArgoCD, which is installed via Terraform in the [infra-helm repo](https://github.com/kolyaiks/argo-deployment-demo-infra-helm). Once all applications are deployed, two versions of the [demo application](https://github.com/kolyaiks/argo-deployment-demo-app) (dev and prod) are exposed via HTTPS through an AWS Application Load Balancer. Each application displays an environment name sourced from AWS Secrets Manager, injected into the pod as an environment variable, and the application version pulled from the [`pom.xml`](https://github.com/kolyaiks/argo-deployment-demo-app/blob/main/pom.xml#L9) at build time via Maven resource filtering. Changes to the dev environment are synced automatically, while the prod environment requires manual sync.

Here is the current repo tree:

```terminaloutput
.
├── argocd-apps
│   ├── aws-load-balancer-controller.yaml
│   ├── demo-app-dev.yaml
│   ├── demo-app-prod.yaml
│   ├── external-gateway.yaml
│   ├── gateway-api-crds.yaml
│   ├── secrets-store-csi-driver-provider-aws.yaml
│   └── secret-store-csi-driver.yaml
├── platform
│   ├── aws-load-balancer-controller
│   │   └── values.yaml
│   ├── external-gateway
│   │   ├── aws-lbconfig.yaml
│   │   ├── gateway-api-gatewayclass.yaml
│   │   ├── gateway-api-gateway.yaml
│   │   └── namespace.yaml
│   └── gateway-api-crds
│       └── standard-install.yaml
├── README.md
└── workloads
    └── argo-deployment-demo-app
        ├── Chart.yaml
        ├── templates
        │   ├── aws-tgconfig.yaml
        │   ├── deployment.yaml
        │   ├── httproute.yaml
        │   ├── ns.yaml
        │   ├── sa.yaml
        │   ├── secrets-provider-class.yaml
        │   └── svc.yaml
        ├── values-dev.yaml
        └── values-prod.yaml
```


### argocd-apps

This is the folder that contains all the ArgoCD apps that are being picked up by this [bootstrap application](https://github.com/kolyaiks/argo-deployment-demo-infra-helm/blob/main/aws/iac/argocd.tf#L15) in the infra repo, following the app-of-apps ArgoCD pattern. The sync order is controlled via the `argocd.argoproj.io/sync-wave` annotation. ArgoCD waits for each wave to become healthy before proceeding to the next.

1. **`secrets-store-csi-driver`** (wave `-3`, chart `secrets-store-csi-driver` 1.6.0) — Installs the Secrets Store CSI driver into `kube-system` with `syncSecret.enabled=true` so it can create Kubernetes Secrets from AWS Secrets Manager. Automated sync.

2. **`secrets-store-csi-driver-provider-aws`** (wave `-2`, chart `secrets-store-csi-driver-provider-aws` 3.1.1) — Installs the AWS provider DaemonSet for the CSI driver. Has `secrets-store-csi-driver.install=false` since the driver is managed by the app above. Automated sync.

3. **`gateway-api-crds`** (wave `-1`, branch `main`) — Installs the standard Gateway API CRDs (v1.5.0) from `platform/gateway-api-crds/standard-install.yaml`. File with CRDs comes from [here](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/gateway/gateway/#prerequisites). Automated sync.

4. **`aws-load-balancer-controller`** (wave `0`, chart `aws-load-balancer-controller` 3.4.0 + branch `main` for values) — Installs the AWS Load Balancer Controller from the EKS Helm chart with cluster/region/VPC settings and IAM role annotation. Values from `platform/aws-load-balancer-controller/values.yaml`. Automated sync.

5. **`external-gateway`** (wave `1`, branch `main`) — Creates the GatewayClass, internet-facing ALB Gateway, LoadBalancerConfiguration, and the `platform` namespace from `platform/external-gateway/`. Automated sync.

6. **`demo-app-dev`** (wave `2`, branch **`dev`**) — Deploys the demo application to the `dev` namespace via `workloads/argo-deployment-demo-app` with `values-dev.yaml`. Automated sync. The CI pipeline in the [kolyaiks/argo-deployment-demo-app](https://github.com/kolyaiks/argo-deployment-demo-app/blob/main/.github/workflows/update_dev_helm.yml) pushes to the `dev` branch of this repo, which triggers ArgoCD sync.

7. **`demo-app-prod`** (wave `2`, branch `main`) — Deploys the demo application to the `prod` namespace via `workloads/argo-deployment-demo-app` with `values-prod.yaml`. **Manual sync only.**

### external-gateway

This is the app that provisions the ALB in AWS. Based on [this aws article](https://repost.aws/articles/ARwuFtb4dlQm-6PhHYH8zezQ/implementing-kubernetes-gateway-api-using-aws-load-balancer-controller-part-ii-l7-albgatewayapi) we create this list of resources:

1. **`namespace.yaml`** (wave `-2`) — Creates the `platform` namespace that hosts all gateway-related resources, keeping them isolated from application namespaces.

2. **`aws-lbconfig.yaml`** (wave `-1`) — A `LoadBalancerConfiguration` resource that sets `scheme: internet-facing`. Without this, the ALB defaults to internal (private) and would only be reachable from within the VPC.

3. **`gateway-api-gatewayclass.yaml`** (wave `1`) — A `GatewayClass` named `alb` with `controllerName: gateway.k8s.aws/alb`. This is a cluster-scoped resource that tells the AWS Load Balancer Controller which Gateway API controller should handle Gateways referencing this class.

4. **`gateway-api-gateway.yaml`** (wave `2`) — The `Gateway` resource that actually provisions the internet-facing ALB. It references the `alb` GatewayClass, attaches to the LoadBalancerConfiguration from step 2, and defines an HTTPS listener on port 443 that accepts routes from all namespaces.

> **Note:** The ALB listener itself won't be provisioned in AWS until an HTTPRoute (from `workloads/argo-deployment-demo-app/templates/httproute.yaml`) references this Gateway. The Gateway definition only declares the listener; the route binding is what triggers the actual target group and listener creation on the ALB.
>
> In addition, `workloads/argo-deployment-demo-app/templates/aws-tgconfig.yaml` is required to make the ALB work with `ClusterIP` services. Without it, the AWS Load Balancer Controller defaults to **Instance** target type, which expects `NodePort` or `LoadBalancer` services. This `TargetGroupConfiguration` resource sets `targetType: ip`, instructing the controller to route traffic directly to pod IPs instead of node ports. It is templated with `{{ .Values.namespace }}` so each environment (dev, prod, etc.) gets its own configuration scoped to its service.

### argo-deployment-demo-app

This is the folder that contains the application Helm chart, parametrised by per-environment values files. The chart is deployed by `demo-app-dev` and `demo-app-prod` ArgoCD apps. Each template is described below:

1. **`ns.yaml`** (wave `-2`) — Creates the target namespace (e.g. `dev` or `prod`) before any other resources in the chart are applied.

2. **`sa.yaml`** — Creates a `ServiceAccount` named `app-sa` annotated with the IAM role ARN (`eks.amazonaws.com/role-arn`). This allows the pod to assume the role and pull secrets from AWS Secrets Manager. The IAM roles are created in the [infra-helm repo](https://github.com/kolyaiks/argo-deployment-demo-infra-helm) — [dev role](https://github.com/kolyaiks/argo-deployment-demo-infra-helm/blob/main/aws/iac/secrets_store_csi_driver.tf#L23) and [prod role](https://github.com/kolyaiks/argo-deployment-demo-infra-helm/blob/main/aws/iac/secrets_store_csi_driver.tf#L74).

3. **`secrets-provider-class.yaml`** (wave `-2`) — A `SecretProviderClass` that configures the Secrets Store CSI driver to fetch a secret from AWS Secrets Manager (`{{ .Values.namespace }}/env`), parse a key-pair value via jmesPath, and create a Kubernetes Secret called `app-k8s-secret` from the result. The underlying secrets in AWS Secrets Manager are created in the [infra-helm repo](https://github.com/kolyaiks/argo-deployment-demo-infra-helm) — [dev secret](https://github.com/kolyaiks/argo-deployment-demo-infra-helm/blob/main/aws/iac/secrets.tf#L1) and [prod secret](https://github.com/kolyaiks/argo-deployment-demo-infra-helm/blob/main/aws/iac/secrets.tf#L10).

4. **`deployment.yaml`** (wave `2`) — The application Deployment. It references `app-sa` as its service account, mounts the CSI volume that triggers the secret creation, and sets the `ENVIRONMENT` environment variable from the `app-k8s-secret` Kubernetes Secret. This environment variable is used by the application to render the environment name in the [HTML page](https://github.com/kolyaiks/argo-deployment-demo-app/blob/main/src/main/resources/templates/index.html#L7). Includes pod anti-affinity to spread replicas across nodes.

5. **`svc.yaml`** — A `ClusterIP` Service that exposes the application on port 80, forwarding to port 8080 in the pod. Annotated with `gateway.k8s.aws/target-type: ip` to inform the AWS Load Balancer Controller to use IP target type.

6. **`httproute.yaml`** — An `HTTPRoute` that binds to the `external-alb-gw` Gateway in the `platform` namespace (HTTPS listener). Routes traffic to the service based on the hostname (e.g. `dev-app.niks.cloud`). Also annotated with `gateway.k8s.aws/target-type: ip`.

7. **`aws-tgconfig.yaml`** — A `TargetGroupConfiguration` that sets `targetType: ip` for the service's target group, required for `ClusterIP` services to work with the AWS Load Balancer Controller (see note in the `external-gateway` section above).

**Values files:**

- **`values-dev.yaml`** — Parameters for the `dev` environment: namespace `dev`, image tag `0.0.8`, hostname `dev-app.niks.cloud`, and IAM role for dev secrets access.
- **`values-prod.yaml`** — Parameters for the `prod` environment: namespace `prod`, image tag `0.0.18`, hostname `prod-app.niks.cloud`, and IAM role for prod secrets access.
