# EKS Hybrid Nodes proof-of-concept (PoC) template

This repository provides a proof-of-concept (PoC) template for **Amazon EKS
Hybrid Nodes**. It provisions a new EKS cluster with **EKS Auto Mode** and
**EKS Hybrid Nodes** enabled, bootstraps and registers on-premises hosts as
hybrid nodes, deploys CoreDNS and kube-proxy as managed add-ons tuned for
hybrid nodes (with topology-aware, same-zone DNS routing), and installs
**Cilium** as the CNI, with a choice of two hybrid Kubernetes
networking options: **Cilium BGP** or the **EKS Hybrid Nodes gateway**.

> **Disclaimer:** This sample streamlines a proof-of-concept deployment of
> Amazon EKS Hybrid Nodes. It is provided for demonstration purposes only and
> should be thoroughly reviewed for security, compliance, and cost implications
> before any production use.

---

## Layout

```
sample-eks-hybrid-nodes-poc/
├── cluster-config.yaml        # 1. eksctl ClusterConfig (Auto Mode + hybrid)
├── hybrid-nodes-creds/        # 2. SSM credentials + node join
│   ├── cfn-ssm-parameters.json#     params for the public IAM-role CFN template
│   ├── access-entry-validation-policy.json  # eks:ListAccessEntries fix (2c.1)
│   └── nodeConfig.yaml        #     nodeadm join config (SSM activation)
├── coredns-addon-config.json  # 3. CoreDNS managed add-on configurationValues
│                              #    (mixed-mode placement + PreferSameZone)
├── cilium-values.yaml         # 4. Cilium Helm values (AWS build; BGP enabled)
│
├── cilium-bgp/                # 5a. OPTION A  -  Cilium BGP CRs (feature on in base)
│   ├── cilium-bgp-cluster.yaml#     BGP instance + peer (localASN/peer)
│   ├── cilium-bgp-peer.yaml   #     timers, graceful restart, address families
│   ├── cilium-bgp-adv-pod.yaml#     advertise PodCIDRs (pick this and/or the LB one)
│   ├── cilium-bgp-adv-lb.yaml #     advertise LoadBalancer service VIPs
│   └── cilium-lb-ippool.yaml  #     LoadBalancer IP pool (Cilium LB IPAM)
│
└── hybrid-nodes-gateway/      # 5b. OPTION B  -  Hybrid Nodes Gateway (HGW, GA)
    ├── gateway-nodepool.yaml  #     Karpenter NodeClass + NodePool (Auto Mode)
    ├── gateway-mng.yaml       #     ALT: managed node groups, one per AZ
    ├── gateway-iam-policy.json#     route-programming IAM policy (Pod Identity)
    └── gateway-az-spread-patch.yaml #  hard 1-per-AZ topologySpread (HA)
```

**Two hybrid Kubernetes networking options**  -  `cilium-bgp` and `hybrid-nodes-gateway`. 
They operate at different layers and are **not** mutually exclusive.


---

## 0. Prerequisites

Have these ready before you start. 

- **An existing VPC** minimum with 2 public + 2 private subnets, and **private
  connectivity** (Site-to-Site VPN or AWS Direct Connect) between the VPC and
  your on-premises network, with routing in both directions.
- **On-prem hosts** running a [supported OS](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-os.html#_version_compatibility).
- **Two RFC-1918 or CGNAT CIDR blocks** reserved for `RemoteNodeNetwork` and `RemotePodNetwork`.
- **Cluster Security Groups** configured [as per here](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-networking.html#hybrid-nodes-networking-cluster-sg), with **On-prem FW rules** configured [as documented here](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-networking.html#hybrid-nodes-networking-on-prem). 
- **Workstation tools:** `awscli` (configured with credentials that can create
  IAM roles, EKS clusters, and SSM activations), `eksctl`, `kubectl`, `helm`,
  `gettext` (for `envsubst`), and the [cilium CLI](https://github.com/cilium/cilium-cli)
  (for the `cilium bgp` checks in step 5a).


---

## 1. Create an EKS cluster enabled with EKS Auto Mode and EKS Hybrid Nodes

Export the cluster identity and network parameters, then create the cluster.
`eksctl` waits until the control plane is ACTIVE before returning.

```bash
export EKS_CLUSTER_NAME="demo-cluster"
export AWS_REGION="${AWS_REGION:-ap-southeast-2}"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query 'Account' --output text)"
export K8S_VERSION="1.36"
export EKS_CLUSTER_ARN="arn:aws:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/${EKS_CLUSTER_NAME}"

# Existing VPC subnets (from step 0)
export PUBLIC_SUBNET_1="subnet-xxxxxxxx"
export PUBLIC_SUBNET_2="subnet-xxxxxxxx"
export PRIVATE_SUBNET_1="subnet-xxxxxxxx"
export PRIVATE_SUBNET_2="subnet-xxxxxxxx"

# On-prem networks
export REMOTE_NODE_CIDR="<REMOTE_NODE_CIDR>"   # on-prem node network, e.g. 10.0.0.0/24
export REMOTE_POD_CIDR="<REMOTE_POD_CIDR>"     # on-prem pod network (required for webhook & AWS service integrations)

envsubst < cluster-config.yaml | eksctl create cluster -f -
aws eks update-kubeconfig --name "$EKS_CLUSTER_NAME" --region "$AWS_REGION"
```

---

## 2. Prepare SSM credentials and join the on-prem nodes

This example uses **AWS Systems Manager (SSM) hybrid activation** to provision temporary IAM credentials for hybrid nodes registration. The flow is: 
- **(2a)** create an IAM role the nodes will assume 
- **(2b)** create an activation tied to that role, which hands you an activation ID + code 
- **(2c)** tell the cluster to trust that role via an EKS **access entry**
- **(2d)** install `nodeadm` on each host and join the EKS cluster using the activation.


```bash
export ROLE_NAME="AmazonEKSHybridNodesRole"                         # <- changeable
export ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${ROLE_NAME}"
```

### 2a. Create the Hybrid Nodes IAM role (using CloudFormation template)


```bash
cd hybrid-nodes-creds

# Download the official template from the aws/eks-hybrid repo
curl -OL 'https://raw.githubusercontent.com/aws/eks-hybrid/refs/heads/main/example/hybrid-ssm-cfn.yaml'

# Render the parameters and deploy
envsubst < cfn-ssm-parameters.json > cfn-ssm-parameters.rendered.json

aws cloudformation deploy \
   --stack-name "${EKS_CLUSTER_NAME}-hybrid-nodes-role" \
   --template-file hybrid-ssm-cfn.yaml \
   --parameter-overrides file://cfn-ssm-parameters.rendered.json \
   --capabilities CAPABILITY_NAMED_IAM

cd ..

```


### 2b. Create the SSM hybrid activation

```bash
# Capture SSM Activation ID and Code values
read -r SSM_ACTIVATION_ID SSM_ACTIVATION_CODE < <(aws ssm create-activation \
    --region "${AWS_REGION}" \
    --default-instance-name eks-hybrid-nodes \
    --description "Activation for EKS hybrid nodes" \
    --iam-role "${ROLE_NAME}" \
    --tags "Key=EKSClusterARN,Value=${EKS_CLUSTER_ARN}" \
    --registration-limit 10 \
    --query '[ActivationId, ActivationCode]' --output text)
export SSM_ACTIVATION_ID SSM_ACTIVATION_CODE

# Print both once so you can copy them to a safe place.
echo "SSM_ACTIVATION_ID=${SSM_ACTIVATION_ID}"
echo "SSM_ACTIVATION_CODE=${SSM_ACTIVATION_CODE}"
```

> `--registration-limit 10` caps how many hosts can use this activation. Default
> validity is 24 h; add `--expiration-date` to change it.


### 2c. Grant the role cluster access (EKS access entry)

```bash
aws eks create-access-entry \
    --cluster-name "${EKS_CLUSTER_NAME}" \
    --principal-arn "${ROLE_ARN}" \
    --type HYBRID_LINUX
```

### 2c.1. Add the access-entry read permission to the role

> `nodeadm init` runs a pre-flight check that calls `eks:ListAccessEntries` /
> `eks:DescribeAccessEntry` to confirm the access entry from 2c is in place.
> The permissions are documented as required in
> [Prepare credentials for hybrid nodes](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-creds.html).


```bash
envsubst < hybrid-nodes-creds/access-entry-validation-policy.json > aev.rendered.json
aws iam put-role-policy \
    --role-name "${ROLE_NAME}" \
    --policy-name EKSAccessEntryValidation \
    --policy-document file://aev.rendered.json

# verify
aws iam get-role-policy --role-name "${ROLE_NAME}" --policy-name EKSAccessEntryValidation \
    --query 'PolicyDocument.Statement[].[Action,Resource]' --output json
```

### 2d. Install `nodeadm` and join each on-prem host

First render the node config **on the workstation** (it needs the activation
values you captured above), then copy it to each host:

```bash
envsubst < hybrid-nodes-creds/nodeConfig.yaml > nodeConfig.rendered.yaml
scp nodeConfig.rendered.yaml <user>@<onprem-host>:~/nodeConfig.yaml
```

Then **on each on-prem host** (commands need `sudo`/root):

```bash
# 1. Download the hybrid-nodes CLI (pick your architecture)
curl -OL 'https://hybrid-assets.eks.amazonaws.com/releases/latest/bin/linux/amd64/nodeadm'   # x86_64
# curl -OL 'https://hybrid-assets.eks.amazonaws.com/releases/latest/bin/linux/arm64/nodeadm' # ARM
chmod +x nodeadm && sudo mv nodeadm /usr/local/bin/

# 2. Install dependencies (containerd, kubelet, kubectl, SSM agent).
#    Replace <K8S_VERSION> with your cluster's Kubernetes minor version,
#    e.g. 1.36  -  it must match the K8S_VERSION used in step 1.
sudo nodeadm install <K8S_VERSION> --credential-provider ssm

# 3. Join the cluster using the rendered config
sudo nodeadm init -c file://nodeConfig.yaml
```

### 2e. Verify and label the nodes

Back on the workstation:

```bash
kubectl get nodes -o wide
```

The hybrid nodes appear with SSM-generated names (`mi-0123...`) and status
**`NotReady`**  -  that's expected until Cilium is installed in step 4. Label them
so the CoreDNS and BGP selectors in later steps match:

```bash
kubectl label node <mi-xxxxxxxx> topology.kubernetes.io/zone=onprem --overwrite
```

---

## 3. Install kube-proxy + CoreDNS (managed add-ons, mixed-mode)

The cluster was created with `disableDefaultAddons: true` (step 1), since they are not required for EKS Auto Mode managed cloud nodes. 
However, we'll provision CoreDNS and kube-proxy for the hybrid nodes as **EKS managed add-ons**,
so you get AWS-managed version upgrades and add-on health reporting in the console.


```bash
# kube-proxy first 
aws eks create-addon \
    --cluster-name "${EKS_CLUSTER_NAME}" \
    --addon-name kube-proxy

# CoreDNS with mixed-mode placement + same-zone routing
aws eks create-addon \
    --cluster-name "${EKS_CLUSTER_NAME}" \
    --addon-name coredns \
    --configuration-values file://coredns-addon-config.json \
    --resolve-conflicts OVERWRITE

# Watch both add-ons (CoreDNS pods stay Pending until Cilium is up)
aws eks list-addons --cluster-name "${EKS_CLUSTER_NAME}"
aws eks describe-addon --cluster-name "${EKS_CLUSTER_NAME}" --addon-name coredns \
    --query 'addon.status'
```

> **Prereq for same-zone DNS:** the `topology.kubernetes.io/zone=onprem` label
> from step 2e must be on the hybrid nodes, and Cilium (step 4) must have
> `serviceTopology: true` (it does)  -  otherwise `PreferSameZone` has no zone to
> match and silently falls back to cluster-wide routing.


## 4. Install Cilium (AWS-maintained build)

[Cilium is the AWS-supported CNI for hybrid nodes](https://aws.amazon.com/about-aws/whats-new/2025/08/expanded-support-cilium-amazon-eks-hybrid-nodes/). 
Install the [AWS-maintained Cilium build](https://gallery.ecr.aws/eks/cilium/cilium) via public ECR repo so the CNI is covered by AWS support. [Mind the kernel floor](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-cni.html#hybrid-nodes-cilium-version-compatibility): 1.18.3+ need Linux kernel ≥ 5.10 (No Ubuntu 20.04 or RHEL 8).

```bash
export POD_CIDR="<POD_CIDR>"           # must equal REMOTE_POD_CIDR
export POD_CIDR_MASK_SIZE="26"
export CILIUM_VERSION="1.19.4-1"        # AWS-maintained ECR tag

envsubst < cilium-values.yaml > /tmp/cilium-values.yaml

helm install cilium oci://public.ecr.aws/eks/cilium/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  -f /tmp/cilium-values.yaml
```

Once Cilium is running, the hybrid nodes flip to **`Ready`** and the CoreDNS
replicas from step 3 finish scheduling (including the on-prem one).


## 5a. OPTION A  -  Cilium BGP Control Plane

The BGP control plane is already enabled by `cilium-values.yaml` (step 4); it
just needs the CRs below to peer and advertise. 

```bash
export BGP_INSTANCE_NAME="<BGP_INSTANCE_NAME>"   # e.g. rack0
export BGP_LOCAL_ASN="<BGP_LOCAL_ASN>"           # ASN for the Cilium nodes
export BGP_PEER_NAME="<BGP_PEER_NAME>"
export BGP_PEER_ASN="<BGP_PEER_ASN>"             # ASN of your on-prem router
export BGP_PEER_ADDRESS="<BGP_PEER_ADDRESS>"   # on-prem router IP

# Peering (always needed). cilium-bgp-peer.yaml has no variables, so apply it
# directly; cilium-bgp-cluster.yaml needs the BGP_* vars rendered in first.
kubectl apply -f cilium-bgp/cilium-bgp-peer.yaml
envsubst < cilium-bgp/cilium-bgp-cluster.yaml | kubectl apply -f -

cilium bgp peers          # verify Session State = established
```

**Then choose what to advertise**  -  pick either pod/svc or both:

```bash
# (a) Advertise pod CIDRs  -  makes on-prem pod networks routable
kubectl apply -f cilium-bgp/cilium-bgp-adv-pod.yaml

# (b) Advertise LoadBalancer service VIPs  -  for Services exposed on-prem
export LB_POOL_CIDR="<LB_POOL_CIDR>"           # LoadBalancer IP range, e.g. 10.0.1.0/28
envsubst < cilium-bgp/cilium-lb-ippool.yaml | kubectl apply -f -
kubectl apply -f cilium-bgp/cilium-bgp-adv-lb.yaml
```

> See [Configure Cilium BGP](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-cilium-bgp.html)
> and [Services of type LoadBalancer](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-load-balancing.html)
> for the full walkthrough.

## 5b. OPTION B  -  Hybrid Nodes Gateway

The Hybrid Nodes Gateway carries hybrid-pod<->region traffic over a
VXLAN/VTEP datapath. Prereq: UDP **8472** allowed both in the gateway nodes'
security group and your on-prem firewall.

**1) Enable Cilium VTEP.** The gateway needs VTEP enabled on Cilium; this is what
registers the `CiliumVTEPConfig` CRD. 
Enable VTEP and disable the L7 proxy on the existing Cilium install, then restart the DaemonSet:

```bash
# CILIUM_VERSION from step 4; VTEP needs Cilium >= 1.17.13-1 / 1.18.8-1 / 1.19.2-1
helm upgrade cilium oci://public.ecr.aws/eks/cilium/cilium \
  --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --reuse-values \
  --set vtep.enabled=true \
  --set l7Proxy=false

# Restart BOTH the operator and the agent. 
kubectl rollout restart deployment/cilium-operator -n kube-system
kubectl rollout status  deployment/cilium-operator -n kube-system
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout status  daemonset/cilium -n kube-system

# verify VTEP on + L7 proxy off, and the CRD is now registered
kubectl get configmap cilium-config -n kube-system -o yaml | grep -E "enable-vtep|enable-l7-proxy"
kubectl get crd ciliumvtepconfigs.cilium.io
```


**2) Provision a Karpenter NodePool for gateway nodes (across 2 AZs).** 

```bash
export AUTO_MODE_NODE_ROLE=$(aws iam list-roles \
  --query "Roles[?contains(RoleName,'${EKS_CLUSTER_NAME}') && contains(RoleName,'AutoModeNodeRole')].RoleName" \
  --output text | head -1)

# sanity check - all four must be non-empty and single-line before rendering
printf 'cluster=%s\nrole=%s\nsubnets=%s %s\n' \
  "$EKS_CLUSTER_NAME" "$AUTO_MODE_NODE_ROLE" "$PRIVATE_SUBNET_1" "$PRIVATE_SUBNET_2"

# Karpenter NodePool (Auto Mode, recommended). Selects gen5+ c/m/r instances
# and pins nodes to PRIVATE_SUBNET_1/2 for two-AZ HA.
envsubst < hybrid-nodes-gateway/gateway-nodepool.yaml | kubectl apply -f -

```

> **Not using EKS Auto Mode?** Apply [`gateway-mng.yaml`](hybrid-nodes-gateway/gateway-mng.yaml)
> instead of the Karpenter NodePool above - it provisions fixed managed node groups,
> one per AZ. See the file header for the source/dest-check requirement, and add
> `--set autoMode.enabled=false` to the gateway Helm install in step 4.


**3) Grant VPC route-table permissions via EKS Pod Identity (recommended).** 

```bash
# Pod Identity agent (skip if already installed)
aws eks create-addon --cluster-name "${EKS_CLUSTER_NAME}" --addon-name eks-pod-identity-agent

# Role trusted by pods.eks.amazonaws.com, with the route-programming policy
aws iam create-role --role-name EKSHybridNodesGatewayRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}'
aws iam put-role-policy --role-name EKSHybridNodesGatewayRole \
  --policy-name HybridNodesGatewayRouteTable \
  --policy-document file://hybrid-nodes-gateway/gateway-iam-policy.json

aws eks create-pod-identity-association \
  --cluster-name "${EKS_CLUSTER_NAME}" \
  --namespace eks-hybrid-nodes-gateway \
  --service-account eks-hybrid-nodes-gateway \
  --role-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/EKSHybridNodesGatewayRole"
```

**4) Install the Hybrid Nodes gateway** 
The chart needs three required values: `vpcCIDR`, `podCIDRs`, `routeTableIDs`. 

```bash
export GATEWAY_CHART_VERSION="1.0.1"    # latest in ECR: gallery.ecr.aws/eks/eks-hybrid-nodes-gateway

# VPC ID -> VPC CIDR (auto-discover from the cluster)
VPC_ID=$(aws eks describe-cluster --name "${EKS_CLUSTER_NAME}" --region "${AWS_REGION}" \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)
export VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids "${VPC_ID}" --region "${AWS_REGION}" \
  --query 'Vpcs[0].CidrBlock' --output text)

# Pod CIDR(s) Cilium hands out on hybrid nodes - same value as step 1
export POD_CIDRS="${REMOTE_POD_CIDR}"

# Route tables to program with hybrid pod routes. Grab ALL route tables in the
# VPC (both private AND public) - this enables webhook and AWS service integrations.

export ROUTE_TABLE_IDS=$(aws ec2 describe-route-tables --region "${AWS_REGION}" \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query 'RouteTables[].RouteTableId' --output text | tr '\t' ',' | sed 's/,/\\,/g')

# sanity check - none of these may be empty
printf 'chart=%s\nvpcCIDR=%s\npodCIDRs=%s\nrouteTableIDs=%s\n' \
  "$GATEWAY_CHART_VERSION" "$VPC_CIDR" "$POD_CIDRS" "$ROUTE_TABLE_IDS"

helm install eks-hybrid-nodes-gateway \
  oci://public.ecr.aws/eks/eks-hybrid-nodes-gateway \
  --version "${GATEWAY_CHART_VERSION}" \
  --namespace eks-hybrid-nodes-gateway \
  --create-namespace \
  --set vpcCIDR="${VPC_CIDR}" \
  --set podCIDRs="${POD_CIDRS}" \
  --set routeTableIDs="${ROUTE_TABLE_IDS}"
  # add --set autoMode.enabled=false if using gateway-mng.yaml nodes
```


**5) Verify**
two hybrid gateway pods running, with the primary/active gateway pod holding the leader lease, and VPC routes
for the hybrid-pod CIDRs pointing at the leader ENI:

```bash
kubectl -n eks-hybrid-nodes-gateway get pods
kubectl -n eks-hybrid-nodes-gateway get lease hybrid-gateway-leader
```

> **Multi-AZ placement** The gateway deployment ships **hard** hostname anti-affinity 
> plus **soft** zone anti-affinity (`preferredDuringScheduling`, `weight:
> 100` on `topology.kubernetes.io/zone`)  -  the two pods *prefer* different AZs out
> of the box but this is only a **preference**: if a second AZ can't place a node
> at scheduling time, the soft rule silently co-locates both gateway pods in the same AZ.
>
> For a **hard** one-per-AZ guarantee, apply `gateway-az-spread-patch.yaml` after
> the install (it adds a `DoNotSchedule` zone `topologySpreadConstraints`, forcing
> Karpenter to build in the second AZ):
> ```bash
> kubectl -n eks-hybrid-nodes-gateway patch deployment eks-hybrid-nodes-gateway \
>   --type=strategic --patch-file hybrid-nodes-gateway/gateway-az-spread-patch.yaml
> ```
> Confirm the two gateway pods land in different AZs:
> ```bash
> for p in $(kubectl -n eks-hybrid-nodes-gateway get pods -o name); do
>   node=$(kubectl -n eks-hybrid-nodes-gateway get "$p" -o jsonpath='{.spec.nodeName}')
>   echo "$p -> $(kubectl get node "$node" -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}')"
> done
> ```

To run end-to-end networking tests for the Hybrid Nodes Gateway (pod-to-pod and
pod-to-EC2 across environments, plus gateway failover), see the **Testing**
section in this post:
[Simplify hybrid Kubernetes networking with Amazon EKS Hybrid Nodes gateway](https://aws.amazon.com/blogs/containers/simplify-hybrid-kubernetes-networking-with-amazon-eks-hybrid-nodes-gateway/)

---

## References

- [Amazon EKS Hybrid Nodes overview](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-overview.html)
- [A deep dive into Amazon EKS Hybrid Nodes](https://aws.amazon.com/blogs/containers/a-deep-dive-into-amazon-eks-hybrid-nodes/)
- [Deep dive into cluster networking for Amazon EKS Hybrid Nodes](https://aws.amazon.com/blogs/containers/deep-dive-into-cluster-networking-for-amazon-eks-hybrid-nodes/)
- [Simplify hybrid Kubernetes networking with Amazon EKS Hybrid Nodes gateway](https://aws.amazon.com/blogs/containers/simplify-hybrid-kubernetes-networking-with-amazon-eks-hybrid-nodes-gateway/)

---

## License

This project is licensed under the MIT-0 License - see the [LICENSE](LICENSE) file.
