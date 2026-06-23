# EKS Cluster Setup — AWS

> **FR** — Mise en place complète d'un cluster Amazon EKS depuis la console AWS : configuration réseau (VPC CloudFormation), rôles IAM, add-ons cluster, node groups managés, autoscaling automatique via Cluster Autoscaler (OIDC/IRSA), déploiement sur Fargate, et provisionnement en une commande avec eksctl.
>
> **EN** — End-to-end setup of an Amazon EKS cluster from the AWS console: VPC networking (CloudFormation), IAM roles, cluster add-ons, managed node groups, automatic scaling with Cluster Autoscaler (OIDC/IRSA), Fargate serverless pods, and one-command provisioning with eksctl.

---

## Stack

![AWS](https://img.shields.io/badge/AWS-EKS%20%7C%20IAM%20%7C%20EC2%20%7C%20Fargate-orange?logo=amazonaws)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.35-326CE5?logo=kubernetes)
![eksctl](https://img.shields.io/badge/eksctl-CLI-FF9900)
![CloudFormation](https://img.shields.io/badge/CloudFormation-VPC-FF4F8B?logo=amazonaws)

---

## FR — Description

### Partie 1 — Création du cluster depuis la console

**Ressources créées :** IAM Role pour EKS, VPC CloudFormation (subnets publics + privés), cluster EKS, node group managé EC2 (`t3.small`), kubeconfig local

**Concepts démontrés :**
- Rôle IAM dédié au control plane EKS : permet à AWS de provisionner les composants du cluster (load balancers, security groups) en votre nom
- VPC spécifique à EKS via template CloudFormation officiel : séparation subnets publics / privés, communication control plane ↔ worker nodes
- Options d'accès à l'API server : Public, Private, Public+Private — compromis entre sécurité et accessibilité
- Add-ons cluster : CoreDNS, kube-proxy, VPC CNI, Metrics Server, EKS Pod Identity Agent
- Node group : kubelet, kube-proxy et container runtime installés automatiquement sur les EC2 workers

### Partie 2 — Cluster Autoscaler

**Ressources créées :** IAM Policy + IAM Role (Web Identity / OIDC), OIDC provider, déploiement Cluster Autoscaler dans `kube-system`

**Concepts démontrés :**
- Problématique du bridge AWS ↔ Kubernetes : les Service Accounts Kubernetes ne sont pas des entités IAM — OIDC (OpenID Connect) établit la confiance entre les deux
- IAM Role avec `Web Identity` : permet au Service Account du Cluster Autoscaler d'assumer le rôle et d'appeler l'API AWS (Auto Scaling Group)
- Tags ASG requis pour l'autodiscovery : `k8s.io/cluster-autoscaler/enabled` et `k8s.io/cluster-autoscaler/<cluster-name>`
- Annotation `safe-to-evict: "false"` pour éviter que l'autoscaler s'évince lui-même lors d'un scale-down
- Test scale-up (20 réplicas → 3 nœuds EC2) et scale-down (1 réplica → retour à 1 nœud après ~10 minutes)

### Partie 3 — Fargate

**Ressources créées :** IAM Role Fargate pod execution, Fargate Profile `dev-profile` avec namespace + label selectors, namespace Kubernetes `dev`

**Concepts démontrés :**
- Fargate vs Node Group : 1 pod = 1 VM isolée managée par AWS (hors du compte client), pas de DaemonSet ni d'application stateful
- Fargate Profile : règle de sélection de pods — namespace + labels doivent correspondre pour qu'un pod soit schedulé sur Fargate
- Label selectors : filtrage fin à l'intérieur d'un namespace (ex. `profile: fargate`) pour mixer Fargate et EC2 dans le même namespace
- Les VMs Fargate utilisent les subnets privés du VPC via ENI — leurs IPs proviennent de notre VPC même si les VMs sont dans le compte AWS
- Validation avec `kubectl get pod -o wide` : chaque pod `dev` tourne sur un nœud `fargate-ip-*` distinct

### Partie 4 — eksctl

**Concepts démontrés :**
- Un seul `eksctl create cluster` crée VPC, subnets, IAM roles, control plane, node group et kubeconfig automatiquement
- Alternative au YAML config file (`eksctl create cluster -f cluster.yaml`) pour une définition reproductible et versionnée du cluster
- eksctl gère aussi les upgrades, la gestion des node groups et la configuration IAM post-création

---

## EN — Description

### Part 1 — Console Cluster Setup

**Created resources:** EKS IAM Role, CloudFormation VPC (public + private subnets), EKS cluster, managed EC2 node group (`t3.small`), local kubeconfig

**Concepts demonstrated:**
- Dedicated IAM role for the EKS control plane: lets AWS provision cluster components (load balancers, security groups) on your behalf
- EKS-specific VPC via the official CloudFormation template: public/private subnet separation, control plane ↔ worker node communication
- API server access options: Public, Private, Public+Private — security vs. accessibility trade-off
- Cluster add-ons: CoreDNS, kube-proxy, VPC CNI, Metrics Server, EKS Pod Identity Agent
- Node group: kubelet, kube-proxy, and container runtime installed automatically on EC2 workers

### Part 2 — Cluster Autoscaler

**Created resources:** IAM Policy + IAM Role (Web Identity / OIDC), OIDC provider, Cluster Autoscaler deployment in `kube-system`

**Concepts demonstrated:**
- The AWS ↔ Kubernetes trust gap: Kubernetes Service Accounts are not IAM entities — OIDC bridges the two
- IAM Role with `Web Identity`: allows the Cluster Autoscaler Service Account to assume the role and call the AWS Auto Scaling API
- ASG tags required for autodiscovery: `k8s.io/cluster-autoscaler/enabled` and `k8s.io/cluster-autoscaler/<cluster-name>`
- `safe-to-evict: "false"` annotation to prevent the autoscaler from evicting itself during scale-down
- Scale-up test (20 replicas → 3 EC2 nodes) and scale-down (1 replica → back to 1 node after ~10 minutes)

### Part 3 — Fargate

**Created resources:** Fargate pod execution IAM Role, Fargate Profile `dev-profile` with namespace + label selectors, `dev` Kubernetes namespace

**Concepts demonstrated:**
- Fargate vs Node Group: 1 pod = 1 isolated VM managed by AWS (outside the customer account), no DaemonSets or stateful workloads
- Fargate Profile: pod selection rule — namespace + labels must match for a pod to be scheduled on Fargate
- Label selectors: fine-grained filtering within a namespace (e.g. `profile: fargate`) to mix Fargate and EC2 in the same namespace
- Fargate VMs use private VPC subnets via ENI — their IPs come from our VPC even though the VMs run in AWS's account
- Validated with `kubectl get pod -o wide`: each `dev` pod runs on its own distinct `fargate-ip-*` node

### Part 4 — eksctl

**Concepts demonstrated:**
- A single `eksctl create cluster` creates VPC, subnets, IAM roles, control plane, node group, and kubeconfig automatically
- YAML config file alternative (`eksctl create cluster -f cluster.yaml`) for a reproducible, version-controlled cluster definition
- eksctl also handles post-creation tasks: cluster upgrades, node group management, and IAM configuration

---

## Repository

The detailed step-by-step guide (console screenshots, kubectl outputs, YAML manifests) lives in the source repository:

[k8s-eks-cluster-setup](https://github.com/m-bengueddache/k8s-eks-cluster-setup)
