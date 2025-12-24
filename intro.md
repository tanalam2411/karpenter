

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
- Optional: ConsolidateAfter (by default set to 0)

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
```

### Karpenter optimization with ConsolidationPolicy

Without consolidateAfter (by default set to 0)

<img width="690" height="299" alt="image" src="https://github.com/user-attachments/assets/d61e092e-dae3-4b36-8597-61bcbbeb4b70" />

This could cause nodes to be created and terminated frequently.


#### Time base consideration: consolidateAfter with WhenEmptyOrUnderutilized


```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
spec:
  distruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1h
```

- Here with the help of `consolidateAfter: 1h` we've added essentially an hour gap(after last pod creation or deletion) between when each action will take place in terms of consolidation

<img width="1222" height="554" alt="image" src="https://github.com/user-attachments/assets/078062f0-e68e-448f-b735-47a4d519bf97" />

Above example is delete scnario, now will see replace scnario

<img width="922" height="300" alt="image" src="https://github.com/user-attachments/assets/fbcd0ed7-2610-4e16-958b-4fb482b982d4" />

to

<img width="719" height="304" alt="image" src="https://github.com/user-attachments/assets/2994bca7-6eb6-4c61-8052-5bda772935bc" />


## Correcting nodes experiencing Drift

> Drift: detects and corrects NodeClaim's which no longer match their owning NodePool, and/or NodeClass specifications

- Carpenter ends up going and provisioning a node claim in association with every single node that it ends up creating
- Periodically karpenter will check if node claim are in desired state along with your node-pool and node-class
- 

```yaml
kind: NodePool
...
spec:
  template:
    spec:
      requirements:
      - key: "karpenter.sh/capacity-type
        operator: In
        values: ["spot"]
```

```yaml
kind: NodePool
...
spec:
  template:
    spec:
      requirements:
      - key: "karpenter.sh/capacity-type
        operator: In
        values: ["on-demand"]
```


## Karpenter Drift & Disruption Summary

### Drift in Karpenter
- Karpenter provisions nodes based on **NodePool** and **NodeClass**.
- Each node has an associated **NodeClaim**.
- Karpenter continuously checks if NodeClaims match the desired state.
- If a mismatch is detected, the node is marked as **drifted**.

### What happens when a node drifts
- The node is **tainted** to stop new workloads.
- Karpenter simulates removing the node.
- If workloads can’t be rescheduled, a **new node is pre-provisioned**.
- The drifted node is **safely drained and removed**.

### Example: Spot → On-Demand
- Change capacity type in NodePool from **Spot to On-Demand**.
- Existing Spot nodes become **drifted**.
- Karpenter replaces them with **On-Demand nodes** safely.

<img width="942" height="478" alt="image" src="https://github.com/user-attachments/assets/b27440ac-a948-43e5-aa01-09336d44daef" />

### Disruption Budgets
- Control **how much**, **when**, and **why** disruption can happen.
- Example rules:
  - **Business hours**:  
    - 0% disruption for **drifted** and **underutilized** nodes.
  - **Any time**:  
    - 100% disruption allowed for **empty** nodes.
  - **General case**:  
    - 10% disruption allowed for **drifted** and **underutilized** nodes.

### Conflict handling
- If disruption budgets conflict,  
  **Karpenter always applies the most restrictive rule**.






