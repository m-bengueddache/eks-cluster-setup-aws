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

IAM Role control plane, VPC via template CloudFormation officiel (subnets publics + privés), cluster EKS, node group managé `t3.small`, kubeconfig local.

Le rôle IAM du control plane est distinct de celui des workers — il donne à AWS les droits de provisionner les composants du cluster (load balancers, security groups) en votre nom. Le VPC utilise le template CloudFormation officiel EKS plutôt que le VPC par défaut, pour la séparation réseau et les tags requis par EKS. L'API server propose trois modes d'accès (Public / Private / Public+Private) avec des compromis sécurité / opérabilité différents. Add-ons activés : CoreDNS, kube-proxy, VPC CNI, Metrics Server, EKS Pod Identity Agent.

### Partie 2 — Cluster Autoscaler

IAM Policy + IAM Role Web Identity (OIDC), OIDC provider, déploiement Cluster Autoscaler dans `kube-system`.

Le problème central : les Service Accounts Kubernetes ne sont pas des entités IAM, OIDC sert de pont entre les deux. Le rôle Web Identity permet au Service Account de l'autoscaler d'appeler l'API Auto Scaling Group. Deux tags doivent être présents sur le ASG pour l'autodiscovery : `k8s.io/cluster-autoscaler/enabled` et `k8s.io/cluster-autoscaler/<cluster-name>`. L'annotation `safe-to-evict: "false"` empêche l'autoscaler de s'évincer lui-même lors d'un scale-down. Testé : 20 réplicas → 3 nœuds EC2, puis retour à 1 nœud en ~10 minutes.

### Partie 3 — Fargate

IAM Role pod execution, Fargate Profile `dev-profile` (namespace + label selectors), namespace `dev`.

Fargate remplace les EC2 workers par 1 VM isolée par pod, gérée par AWS en dehors du compte client — pas de DaemonSet, pas de workload stateful. Le Fargate Profile définit les règles de sélection (namespace + labels) : les pods du namespace `dev` ne sont schedulés sur Fargate que si le label correspond (`profile: fargate`). Cela permet de mixer Fargate et EC2 dans le même namespace. Les VMs Fargate utilisent les subnets privés du VPC via ENI — leurs IPs viennent du CIDR du VPC même si les VMs tournent dans le compte AWS.

### Partie 4 — eksctl

Un seul `eksctl create cluster` crée VPC, subnets, IAM roles, control plane, node group et kubeconfig. Alternative : un fichier YAML config (`eksctl create cluster -f cluster.yaml`) pour une définition reproductible et versionnée. eksctl gère aussi les upgrades, la gestion des node groups et la configuration IAM post-création.

---

## EN — Description

### Part 1 — Console Cluster Setup

IAM Role for the control plane, VPC via the official CloudFormation template (public + private subnets), EKS cluster, managed `t3.small` node group, local kubeconfig.

The control plane IAM role is separate from the worker node role — it lets AWS provision cluster components (load balancers, security groups) on your behalf. The VPC uses the official EKS CloudFormation template rather than the default VPC, for the network separation and subnet tags EKS requires. The API server has three access modes (Public / Private / Public+Private) with different security and operational trade-offs. Add-ons: CoreDNS, kube-proxy, VPC CNI, Metrics Server, EKS Pod Identity Agent.

### Part 2 — Cluster Autoscaler

IAM Policy + IAM Role (Web Identity / OIDC), OIDC provider, Cluster Autoscaler deployment in `kube-system`.

The core challenge: Kubernetes Service Accounts are not IAM entities — OIDC bridges that gap. The Web Identity role lets the autoscaler's Service Account call the AWS Auto Scaling API. Two ASG tags are required for autodiscovery: `k8s.io/cluster-autoscaler/enabled` and `k8s.io/cluster-autoscaler/<cluster-name>`. The `safe-to-evict: "false"` annotation prevents the autoscaler from evicting itself during scale-down. Tested: 20 replicas scaled up to 3 EC2 nodes, then back to 1 node in ~10 minutes.

### Part 3 — Fargate

Fargate pod execution IAM Role, `dev-profile` Fargate Profile (namespace + label selectors), `dev` namespace.

Fargate replaces EC2 worker nodes with 1 isolated VM per pod, managed by AWS outside your account — no DaemonSets, no stateful workloads. The Fargate Profile sets the selection rules (namespace + labels): pods in the `dev` namespace are only scheduled on Fargate when the label matches (`profile: fargate`), which allows mixing Fargate and EC2 pods in the same namespace. Fargate VMs use private VPC subnets via ENI — their IPs come from the VPC CIDR even though the VMs run in AWS's account.

### Part 4 — eksctl

A single `eksctl create cluster` creates VPC, subnets, IAM roles, control plane, node group, and kubeconfig. The YAML config alternative (`eksctl create cluster -f cluster.yaml`) gives a reproducible, version-controlled cluster definition. eksctl also handles post-creation tasks: cluster upgrades, node group management, and IAM configuration.
