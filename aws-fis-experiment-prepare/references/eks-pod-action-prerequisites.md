# EKS Pod Action Prerequisites Reference

## Scope

When the FIS action is any of the following types, you **MUST** complete the prerequisite
steps in this document before generating any configuration files:

- `aws:eks:pod-cpu-stress`
- `aws:eks:pod-delete`
- `aws:eks:pod-io-stress`
- `aws:eks:pod-memory-stress`
- `aws:eks:pod-network-blackhole-port`
- `aws:eks:pod-network-latency`
- `aws:eks:pod-network-packet-loss`

## Official Documentation (Required Reading)

Before generating any configuration files, you **MUST** call `aws___read_documentation` to read:
https://docs.aws.amazon.com/fis/latest/userguide/eks-pod-actions.html

## Prerequisites Checklist

### 1. Kubernetes ServiceAccount + RBAC

Create the following resources in the target namespace:

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  namespace: {TARGET_NAMESPACE}
  name: {SA_NAME}  # Must match kubernetesServiceAccount parameter in experiment template
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: {TARGET_NAMESPACE}
  name: fis-experiment-role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "create", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "list", "get", "delete", "deletecollection"]
- apiGroups: [""]
  resources: ["pods/ephemeralcontainers"]
  verbs: ["update"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: fis-experiment-role-binding
  namespace: {TARGET_NAMESPACE}
subjects:
- kind: ServiceAccount
  name: {SA_NAME}
  namespace: {TARGET_NAMESPACE}
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: fis-experiment
roleRef:
  kind: Role
  name: fis-experiment-role
  apiGroup: rbac.authorization.k8s.io
```

**Important notes:**
- Use namespace-scoped Role, NOT ClusterRole
- RoleBinding must bind BOTH the ServiceAccount AND User `fis-experiment`
- `fis-experiment` is the username specified in the EKS Access Entry

### 2. EKS Access Entry

The Access Entry in the CFN template **MUST** specify `Username: fis-experiment`:

```yaml
EKSAccessEntry:
  Type: AWS::EKS::AccessEntry
  Properties:
    ClusterName: {CLUSTER_NAME}
    PrincipalArn: !GetAtt FISExperimentRole.Arn
    Username: fis-experiment    # CRITICAL: Must match User in RoleBinding
    Type: STANDARD
    # No AccessPolicies needed - permissions are granted via K8S RBAC
```

**Do NOT:**
- Do NOT bind `AmazonEKSClusterAdminPolicy` (overly permissive and unnecessary)
- Do NOT use namespace-scoped AccessPolicy (use K8S RBAC instead)

### 3. EKS Cluster Authentication Mode

The cluster must use `API_AND_CONFIG_MAP` or `API` authentication mode:
```bash
aws eks describe-cluster --name {CLUSTER} \
  --query 'cluster.accessConfig.authenticationMode'
```

### 4. Pod Security Context

The target Pod's `readOnlyRootFilesystem` must be `false`. All EKS Pod actions will
fail if the root filesystem is read-only.

### 5. Network Action Limitations

The following actions do NOT support AWS Fargate or bridge network mode:
- `aws:eks:pod-network-blackhole-port`
- `aws:eks:pod-network-latency`
- `aws:eks:pod-network-packet-loss`

These actions require the ephemeral container to have root privileges. If the Pod
runs as non-root, you must set securityContext individually for containers in the Pod.

## CFN Service Role Permissions

If deploying via CFN with `--role-arn`, the CFN service role needs these EKS permissions:
```json
{
  "Action": [
    "eks:CreateAccessEntry",
    "eks:DeleteAccessEntry",
    "eks:DescribeAccessEntry",
    "eks:AssociateAccessPolicy",
    "eks:DisassociateAccessPolicy",
    "eks:ListAssociatedAccessPolicies"
  ],
  "Resource": "*"
}
```

## IAM Role Name Length

IAM Role `RoleName` in CFN cannot exceed 64 characters.
When using `!Sub` with `${AWS::StackName}`, estimate total length:
- Stack name: max ~40 characters
- Prefix: keep under 20 characters
- Total: stay under 60 characters

## CFN Deployment Notes

Before deployment, check if the environment requires `--role-arn`:
```bash
# Check if existing FIS-related stacks use a service role
aws cloudformation describe-stacks --region {REGION} \
  --query 'Stacks[?contains(StackName,`fis`)].{Name:StackName,Role:RoleARN}' \
  --output table
```

If a CFN service role exists in the environment, all deploy and delete operations
must include `--role-arn`.

