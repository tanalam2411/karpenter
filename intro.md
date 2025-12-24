

# Karpenter

## Problem

### Limitation of Cluster Autoscaler

- Scales same size of nodes
- Nodes are either under or over utilized
- No option to control size of node while scaling
- No zone control
- Workload might need specific type of disks that might not be available in all zones


```yaml
# old
key: "node.k8sinstance/instance.type
operator: in
values: ["t2.small", "t2.large", "t2.xlarge", "t2.2xlarge"]

key: "topology.k8s.io/zone
operator: in
values: ["us-east-1a", "us-east-1b"]

key: "karpenter.sh/capacity-type"
operator: in
values: ["spot"]
```

- Karpenter is fast because if creates nodes directly and doesnt need to interact or modify node groud etc.
- Pros:
  - Cost optimization
  - Supports diverse workloads including ML and generative AI
  - Helps in upgrade and patching as well
  - Compute flexibility
    - Instance type flexibility
      - Attribute based requirements
        - sizes, families, generations, CPU arch
        - if not list provided: picks from all instance types
        - limits how many VM instances this node pool can provision
        - priorities cost
  - Picks the right instance type based on workload
 

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirement:
      - key: karpenter.k8s.aws/instance-category
        operator: IN
        values: ["c", "m", "r", "t"]  
      - key: karpenter.k8s.aws/instance-size
        operator: NotIn
        values: ["nano", "micro", "small", "medium"]
      - key: karpenter.k8s.aws/instance-hyoervisor
        operator: In
        values: ["nitro"]
      - key: topology.kubernetes.io/zone
        operator: In
        values: ["us-west-2a", "us-west-2b"]
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64", "arm64"]
  limits:
    cpu: 100 # this will allow nodes creation till aggregate no. of vcpu reaches this limit
```

## Karpenter works with k8s scheduling

> Standard k8s pod scheduling mechanisms

- Node selectors
- Node affinity
- Taints and tolerations
- Topology spread

## User defained annotations, labels and taints

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
spec:
  template:
    metadata:
      annotations:
        application/name: "app-a"
      labels:
        team: "team-a"
  spec:
    taints:
    - key: example.com/special-taint
      value: true
      effect: NoSchdule
```

- Nodes provisioned by above nodepool will have the `annotations`, `labels` and `taints` mentioned under spec.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  nodeSelector:
    team: "team-a"
```

## Sample well-known labels added to nodes

| Label                                                     | Example        |
|-----------------------------------------------------------|----------------|
| topology.kubernetes.io/zone                               | us-east-2a     |
| node.kubernetes.io/instance-type                          | g4dn.8xlarge   |
| kubernetes.io/os                                          | linux          |
| kubernetes.io/arch                                        | amd64          |
| karpenter.sh/capacity-type                                | spot           |
| karpenter.k8s.aws/instance-hypervisor                     | nitro          |
| karpenter.k8s.aws/instance-encryption-in-transit-supported| true           |
| karpenter.k8s.aws/instance-category                       | g              |


## Node Disruption - Consolidation

<img width="1182" height="250" alt="image" src="https://github.com/user-attachments/assets/60e9cb26-2e78-47b1-9437-9aff7d73b416" />

- Consolidation: Reducing number of nodes or replacing nodes for optimal bin-packing
- Consolidation Policies: WhenEmpty or WhenEmptyorUnderutilized
- Optional: ConsolidateAfter

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
```

