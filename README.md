# hive-gitops

GitOps repository for the Hive EKS cluster, managed by ArgoCD using the app-of-apps pattern.

- [Layout](#layout)
- [Installation](#installation)
  - [1. Tools](#1-tools)
  - [2. Point kubectl at the EKS cluster](#2-point-kubectl-at-the-eks-cluster)
  - [3. AWS prerequisites](#3-aws-prerequisites)
  - [4. Fill in the environment values](#4-fill-in-the-environment-values)
  - [5. Push to Git](#5-push-to-git)
  - [6. Bootstrap the cluster](#6-bootstrap-the-cluster)
  - [7. Log in to ArgoCD](#7-log-in-to-argocd)
- [Day-2 changes](#day-2-changes)
- [Troubleshooting](#troubleshooting)
- [Teardown](#teardown)
- [Adding a new environment](#adding-a-new-environment)

## Layout

```
bootstrap/
  initial/<env>/        # applied once by hand, in order, to seed a fresh cluster
  app-of-apps/          # root Applications (bootstrap, cluster-core, cluster-traffic, argocd)
    base/
    overlays/<env>/     # sets spec.source.path of every root app to overlays/<env>
components/
  namespaces/           # namespaces needed before ArgoCD exists
  argocd/               # ArgoCD install (upstream manifest + patches)
  cluster-core/         # core add-ons as ArgoCD Applications (AWS Load Balancer Controller)
  cluster-traffic/      # external (internet-facing) ALB routing to the workload service
```

Root apps and sync waves:

| Application     | Wave  | Path                                      |
|-----------------|-------|-------------------------------------------|
| cluster-core    | -1000 | components/cluster-core/overlays/<env>    |
| cluster-traffic | -999  | components/cluster-traffic/overlays/<env> |
| bootstrap       | -997  | bootstrap/app-of-apps/overlays/<env>      |
| argocd          | -996  | components/argocd/overlays/<env>          |

The `bootstrap` app points at its own directory, so ArgoCD manages the root apps too.
Every root app's path ends in `PATCH_ME` in `base/`; the overlay's `replacements`
swaps that segment for the `gitops.hive/overlay-path` annotation.

One internet-facing ALB is created by `cluster-traffic`:

| ALB                | Group     | Listener                   | Routes to                  |
|--------------------|-----------|----------------------------|----------------------------|
| `hive-dev-traffic` | `traffic` | HTTP 8080, no TLS, no host | `hive/hive:80`              |

ArgoCD is not exposed through an ALB. Reach it with `kubectl port-forward` (step 7).
If ArgoCD is exposed later with TLS, give its ingress a separate ALB group: `ssl-redirect`
is exclusive per group, so sharing `traffic` would redirect the 8080 listener to HTTPS as well.

## Installation

The steps below use `dev` as the environment. The shell variables are reused throughout,
so set them once:

```sh
export ENV=prod
export CLUSTER_NAME=<eks-cluster-name>
export AWS_REGION=<region>                  # e.g. eu-west-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export VPC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION \
  --query cluster.resourcesVpcConfig.vpcId --output text)
```

### 1. Tools

| Tool      | Used for                                         |
|-----------|--------------------------------------------------|
| `aws`     | IAM, subnet tags, kubeconfig                     |
| `eksctl`  | OIDC provider and IRSA role (optional, see 3.2)  |
| `kubectl` | bootstrap (`kubectl apply -k` bundles kustomize) |
| `argocd`  | CLI access (optional)                            |

### 2. Point kubectl at the EKS cluster

```sh
aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION
kubectl config current-context   # must print the EKS cluster ARN
kubectl get nodes
```

Check the context before every `apply` below. Applying to the wrong cluster installs
ArgoCD there.

### 3. AWS prerequisites

These are not managed by this repo. If the cluster is built with Terraform, create them there instead.

#### 3.1 IAM OIDC provider

The controller authenticates to AWS through IRSA, which needs the cluster's OIDC provider:

```sh
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --region $AWS_REGION --approve
```

#### 3.2 IAM policy and role for the AWS Load Balancer Controller

The policy version matches the controller version in
`components/cluster-core/base/aws-load-balancer-controller.yaml` (v3.5.0).

```sh
curl -o iam_policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.5.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy-$CLUSTER_NAME \
  --policy-document file://iam_policy.json

# --role-only: the Helm chart creates the ServiceAccount itself, eksctl only creates the role
eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME --region $AWS_REGION \
  --namespace kube-system --name aws-load-balancer-controller \
  --role-name $CLUSTER_NAME-aws-lb-controller \
  --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy-$CLUSTER_NAME \
  --role-only --approve
```

Role ARN for step 4: `arn:aws:iam::$AWS_ACCOUNT_ID:role/$CLUSTER_NAME-aws-lb-controller`

#### 3.3 Subnet tags

The controller discovers subnets by tag. The internet-facing ALB needs at least two public subnets
in different AZs, tagged:

```sh
aws ec2 create-tags --resources <public-subnet-a> <public-subnet-b> \
  --tags Key=kubernetes.io/role/elb,Value=1
```

For internal ALBs later, tag the private subnets with `kubernetes.io/role/internal-elb=1`.

### 4. Fill in the environment values

| File | Field | Value |
|------|-------|-------|
| `bootstrap/app-of-apps/base/*.yaml` | `repoURL` | Git URL of this repo (replace `REPLACE_ME`) |
| `components/cluster-core/overlays/dev/patches/aws-load-balancer-controller/cluster-info.yaml` | `clusterName`, `region`, `vpcId` | `$CLUSTER_NAME`, `$AWS_REGION`, `$VPC_ID` |
| same file | `eks.amazonaws.com/role-arn` | role ARN from 3.2 |
| `components/cluster-traffic/overlays/dev/patches/external-ingress/cluster-info.yaml` | `load-balancer-name` | already `hive-dev-traffic`, change only if needed |

`cluster-traffic` sends traffic to service `hive` (port 80) in namespace `hive`. That service
is created by the Helm chart in the `hive-app` repo, release name `hive`. Deploy the chart into
the `hive` namespace (`make deploy ... NAMESPACE=hive`), because the Makefile defaults to
`hive-<env>`. Until the service exists the ALB returns 503. The ALB health check calls
`/health`, the same path as the pod's readiness probe. The ALB listens on 8080 over plain HTTP with no host rule, so every request
on that port goes to the service.

Check that nothing is left and that everything renders:

```sh
grep -rn "PATCH_ME\|REPLACE_ME" components/*/overlays bootstrap/app-of-apps/base   # should print nothing
for d in bootstrap/initial/$ENV/*/; do kubectl kustomize $d > /dev/null && echo "OK $d"; done
```

### 5. Push to Git

ArgoCD reads the root apps and components from Git, so the filled-in values must be pushed
before step 6.4. The root apps track `HEAD`, which is the repository's default branch.

```sh
git add . && git commit -m "Configure $ENV environment" && git push
```

**Private repository:** ArgoCD needs credentials. Create the secret by hand. Do not commit it.
`.sensitive/` is gitignored for this purpose.

```sh
mkdir -p .sensitive
cat > .sensitive/repo-creds.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: hive-gitops-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/<org>/hive-gitops
  username: git
  password: <github-token-with-read-access>
EOF
```

It is applied in step 6.2, after the `argocd` namespace exists.

### 6. Bootstrap the cluster

Run the steps in order and wait for each check before moving on.

#### 6.1 Namespace

```sh
kubectl apply -k bootstrap/initial/$ENV/00-namespace
```

#### 6.2 ArgoCD

`--server-side` is required: the ArgoCD CRDs are too large for client-side apply.

```sh
kubectl apply -k bootstrap/initial/$ENV/01-argocd --server-side
kubectl wait --for condition=established crd/applications.argoproj.io --timeout=60s
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd rollout status deploy/argocd-repo-server
kubectl -n argocd rollout status statefulset/argocd-application-controller

# private repo only
kubectl apply -f .sensitive/repo-creds.yaml
```

#### 6.3 Cluster core (AWS Load Balancer Controller)

This creates the `aws-load-balancer-controller` Application. It pulls the chart from
`aws.github.io/eks-charts`, so it does not depend on this repo being reachable yet.

```sh
kubectl apply -k bootstrap/initial/$ENV/02-cluster-core
kubectl -n argocd get application aws-load-balancer-controller   # wait for Synced / Healthy
kubectl -n kube-system rollout status deploy/aws-load-balancer-controller
```

#### 6.4 App of apps

From here on ArgoCD manages everything from Git, including itself:

```sh
kubectl apply -k bootstrap/initial/$ENV/03-app-of-apps
kubectl -n argocd get applications
```

Expected result after a minute or two:

```
NAME                           SYNC STATUS   HEALTH STATUS
argocd                         Synced        Healthy
aws-load-balancer-controller   Synced        Healthy
bootstrap                      Synced        Healthy
cluster-core                   Synced        Healthy
cluster-traffic                Synced        Healthy
```

Check the ALB:

```sh
kubectl -n hive get ingress external-ingress   # ADDRESS column shows the ALB DNS name
aws elbv2 describe-load-balancers --region $AWS_REGION --names hive-dev-traffic \
  --query 'LoadBalancers[0].[DNSName,State.Code]' --output text
```

The ALB takes 2–3 minutes to reach the `active` state. Then test it:

```sh
curl -i http://<hive-dev-traffic-dns>:8080/   # 503 until the hive service has ready pods
```

### 7. Log in to ArgoCD

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:80
```

Open `http://localhost:8080` and log in as `admin`. argocd-server runs with `server.insecure: "true"`,
so the port-forward serves plain HTTP.

CLI (it opens its own port-forward):

```sh
argocd login --port-forward --port-forward-namespace argocd --plaintext --username admin
argocd account update-password --port-forward --port-forward-namespace argocd --plaintext
kubectl -n argocd delete secret argocd-initial-admin-secret
```

## Day-2 changes

After bootstrapping, change the cluster only through Git. The root apps run with
`selfHeal: true`, so manual `kubectl` changes to managed resources are reverted.

- Change a value: edit the overlay patch, then commit and push. ArgoCD picks it up within 3 minutes (`timeout.reconciliation`).
- Upgrade ArgoCD: change the version in the `install.yaml` URL in `components/argocd/base/kustomization.yaml`.
- Upgrade the controller: change `targetRevision` in `components/cluster-core/base/aws-load-balancer-controller.yaml`. Also update the IAM policy from 3.2 to the matching version.

## Troubleshooting

| Symptom | Likely cause / check |
|---------|----------------------|
| Ingress has no ADDRESS | `kubectl -n kube-system logs deploy/aws-load-balancer-controller`. Usually missing subnet tags (3.3), a wrong role ARN, or an OIDC provider that is not associated (3.1). |
| `failed calling webhook "vingress.elbv2.k8s.aws"` | The controller is not ready yet. The `cluster-traffic` app retries automatically. Wait, or run `argocd app sync cluster-traffic`. |
| `AccessDenied` in controller logs | The IAM policy is outdated for the controller version. Re-download it with the matching tag (3.2). |
| Traffic ALB returns 503 on 8080 | Service `hive/hive` has no ready endpoints: the chart is not deployed to `hive`, or `/health` fails (it pings the database). Check `kubectl -n hive get endpoints hive`. |
| Traffic ALB times out on 8080 | Check that the ALB security group allows 8080 inbound. The controller opens it to `0.0.0.0/0` by default; `alb.ingress.kubernetes.io/inbound-cidrs` narrows it. |
| Root apps stuck on `repository not found` / `authentication required` | `repoURL` is wrong, or the private repo secret (step 5) is missing. |
| `no matches for kind "Application"` in 6.3 | The ArgoCD CRDs are not established yet. Re-run the `kubectl wait` from 6.2. |

## Teardown

Delete the ALB before deleting the cluster. Otherwise the ALB and its security groups are
left behind and block VPC deletion.

```sh
# stop ArgoCD from recreating the ingress
kubectl -n argocd scale statefulset argocd-application-controller --replicas=0
kubectl delete ingress -n hive external-ingress
# wait until this returns LoadBalancerNotFound
aws elbv2 describe-load-balancers --region $AWS_REGION --names hive-dev-traffic
```

Then delete the cluster and the IAM role and policy from 3.2.

## Adding a new environment

1. Copy these directories from `dev` to the new env name:
   - `bootstrap/initial/dev`
   - `bootstrap/app-of-apps/overlays/dev`
   - `components/argocd/overlays/dev`
   - `components/cluster-core/overlays/dev`
   - `components/cluster-traffic/overlays/dev`
2. Set `gitops.hive/overlay-path` in the new app-of-apps overlay to `overlays/<env>`.
3. Point `bootstrap/initial/<env>/*` at the new overlays.
4. Set an env-specific `load-balancer-name` in the cluster-traffic overlay patch. ALB names are unique per account and region.
5. Follow [Installation](#installation) against the new cluster.
