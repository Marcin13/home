# Role Configuration for EKS Node Group: myr-EKS_EFS_CSI_DriverRole

## Create the IAM Role

### Get OIDC Provider

```bash
aws eks describe-cluster --name <cluster-name> --query "cluster.identity.oidc.issuer" --output text
```

Create the role. You can change `myr-EKS_EFS_CSI_DriverRole` to a different name, but if you do, make sure to change it in later steps too.

```bash
aws iam create-role \
    --role-name myr-EKS_EFS_CSI_DriverRole \
    --assume-role-policy-document file://"trust-policy.json"
```

## Trusted Entities

This role is associated with the EFS CSI driver in the EKS cluster and includes the following trust relationship:

## This is content of  `trust-policy.json`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::539702486876:oidc-provider/oidc.eks.eu-central-1.amazonaws.com/id/A2857D369CE912ABDE453AE2DDE6BFA0"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.eu-central-1.amazonaws.com/id/A2857D369CE912ABDE453AE2DDE6BFA0:sub": "system:serviceaccount:kube-system:efs-csi-controller-sa"
                }
            }
        }
    ]
}
```

### OIDC Provider

#### Get OIDC Provider

```bash
aws eks describe-cluster --name <cluster-name> --query "cluster.identity.oidc.issuer" --output text
```

## Expected Output

```bash
`arn:aws:iam::539702486876:oidc-provider/oidc.eks.eu-central-1.amazonaws.com/id/3A27C1015902027C3170D499E6BB608F`
```

Ensure that this OIDC provider is correctly configured in the IAM service and linked to your EKS cluster.

### Service Account

The role assumes identity from the service account `efs-csi-controller-sa` in the `kube-system` namespace.

## Permissions Policy: myr-EKS_EFS_CSI_Driver_Policy

```bash
aws iam create-policy \
    --policy-name EKS_EFS_CSI_Driver_Policy \
    --policy-document file://iam-policy-example.json
``` 

## The policy which will be add to the role myr-EKS_EFS_CSI_DriverRole and the name of the policy is `EKS_EFS_CSI_Driver_Policy`

### This is the content of `iam-policy-example.json`

```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:DescribeAccessPoints",
                "elasticfilesystem:DescribeFileSystems",
                "elasticfilesystem:DescribeMountTargets",
                "ec2:DescribeAvailabilityZones",
                "elasticfilesystem:DescribeAccessPoints",
                "elasticfilesystem:CreateAccessPoint",
                "elasticfilesystem:DeleteAccessPoint"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:CreateAccessPoint"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "aws:RequestTag/efs.csi.aws.com/cluster": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:TagResource"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "aws:ResourceTag/efs.csi.aws.com/cluster": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": "elasticfilesystem:DeleteAccessPoint",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/efs.csi.aws.com/cluster": "true"
                }
            }
        }
    ]
}
```

### Allowed Services and Actions

The role grants permissions to interact with EFS and EC2 services, with specific limitations:

#### EC2 (Elastic Compute Cloud)

- **Access Level:** Limited to List actions
- **Resources:** All resources

#### EFS (Elastic File System)

- **Access Level:** Limited to List, Read, Write, and Tagging actions
- **Resources:** All resources tagged with:
    - `aws:ResourceTag/efs.csi.aws.com/cluster = true`

## Actions to Complete the Setup

### Update OIDC Provider

- Verify the OIDC provider configuration in the IAM service to ensure it matches the EKS cluster's OIDC details.
- If necessary, re-create or update the OIDC provider in the AWS Management Console.

### Recreate or Update the Role

- If this role was accidentally deleted or modified, it must be manually recreated with the exact same configuration.

### Add the Permissions Policy

- Ensure that the policy `myr-EKS_EFS_CSI_Driver_Policy` is created with the permissions and conditions mentioned above.
- Attach this policy to the role.

### Link Role to EFS CSI Controller

- Confirm that the role is correctly linked to the service account `efs-csi-controller-sa` in the `kube-system` namespace using the Kubernetes ServiceAccount annotation.

## Service Account Configuration we need to create

```bash
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/name: aws-efs-csi-driver
  name: efs-csi-controller-sa
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::539702486876:role/myr-EKS_EFS_CSI_DriverRole
```

## Important Notes

- Ensure that the security group and networking configurations are correctly set up for EFS access.
- Confirm that the IAM role has sufficient permissions for any additional resources required by the EFS CSI driver.
- Review the `aws:ResourceTag` conditions if specific tagging is necessary for resource access.



# References

### [Create an IAM policy and role for Amazon EKS](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/docs/iam-policy-create.md)