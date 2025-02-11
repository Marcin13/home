# This is a file contains cmd i run in terminal

## Update kubeconfig

```bash
aws eks update-kubeconfig --region eu-central-1 --name myr-eks
```

## Aws get user, to check if the user is the one we expect

```bash
aws iam get-user
```

## Verify Configuration

```bash
kubectl get nodes
```

## Add & update Helm Repo

```bash
helm repo add jetstack https://charts.jetstack.io && helm repo update
```

## Install cert-manager

```bash
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace  -f oc-cert-manager-values.yaml
```

## Expected output for cert-manager installation

```bash
NAME: cert-manager
LAST DEPLOYED: Fri Jan 17 09:10:18 2025
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
⚠️  WARNING: `installCRDs` is deprecated, use `crds.enabled` instead.
cert-manager v1.16.3 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/
```

### {[ Uninstall cert-manager ]}

```bash
helm uninstall cert-manager --namespace cert-manager
```

## Verify cert-manager installation

```bash
kubectl get pods --namespace cert-manager
```

## Expected output from cert-manager installation

```bash
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-56d4c7dfb7-zjw9t              1/1     Running   0          9m4s
cert-manager-cainjector-6dc54dcd78-6xnwr   1/1     Running   0          9m4s
cert-manager-webhook-5d74598b49-htv6m      1/1     Running   0          9m4s
```

## If everything is ok, let's install cluster-issuers

```bash
kubectl apply -f cluster-issuers.yaml # one for prod and one for staging
```

## Verify cluster-issuers installation

```bash
kubectl get clusterissuers
```

## Expected output for cluster-issuers

```bash
NAME                     READY   AGE
letsencrypt-production   True    52s
letsencrypt-staging      True    70s
```

## Now we need to install the ingress controller, in this case, we will use nginx

```bash
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace -f oc-nginx-ingress-values.yaml
```

### {[ Uninstall nginx-ingress ]}

```bash
helm uninstall nginx-ingress --namespace ingress-nginx
```

## Verify nginx-ingress installation

```bash
kubectl get pods --namespace ingress-nginx
```

## Expected output for nginx-ingress installation

```bash
NAME                                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-ingress-nginx-controller-69786fcbcf-ns7bz   1/1     Running   0          70s
```

## But the most important, we need to verify that the aws load balancer is created (and is not for free)

```bash
aws elb describe-load-balancers --region eu-central-1
```

## also we can get public ip of the load balancer

```bash
kubectl get svc -n ingress-nginx
```

## or more like DNS name

```bash
 NAME                                               TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)                      AGE
nginx-ingress-ingress-nginx-controller             LoadBalancer   10.100.187.190   a1f89970798f0470994dcbff9157a272-1612871129.eu-central-1.elb.amazonaws.com   80:32072/TCP,443:30113/TCP   8m14s
nginx-ingress-ingress-nginx-controller-admission   ClusterIP      10.100.195.55    <none>                                                                       443/TCP                      8m14s
```

## when we got the public ip, we can create a new CNAME record in the DNS provider.

## and check if the DNS is propagated

```bash
nslookup aws.m4bit.pro
```

## Please remember that the DNS propagation can take some time, so be patient.

```bash
nslookup aws.m4bit.pro
```

## so check if the DNS is pointing to the right IP's

## when you pass this url to the browser, you should see the default nginx page, in my case 404 page not found.

## Finally, we can start deploying our apps

```bash
kubectl apply -f deployment.yaml
```

### `Disclaimer about services and AWS`

### when using LoadBalancer, AWS will create a new load balancer for each service, so be careful with the costs.

### NodePort is a good option for testing, but not for production, you need to manage the ports and the security groups.

### Cluster IP is a good option for internal services, but you need to manage the ingress controller.

## Ingress controller when you create a new ingress, it will create a new rule in the load balancer, so you can have multiple services in the same load balancer.

## install PrestaShop shop

```bash
helm install prestashop bitnami/prestashop --namespace prestashop --create-namespace -f aws-prestashop-values.yaml
```

## There is a problem with the prestashop installation, the pvc can't create the volume, so we need create extra aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver and create the storage class

```bash
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/

helm repo update aws-efs-csi-driver
```

## Check the values for the efs-csi-driver

```bash
helm show values aws-efs-csi-driver/aws-efs-csi-driver > efs-csi-driver-values.yaml
```

## Check and update the values for serviceAccount in the efs-csi-driver-values.yaml

```yaml
# Specifies whether a service account should be created
  serviceAccount:
    create: true
    name: efs-csi-controller-sa
    annotations: 
    ## Enable if EKS IAM for SA is used
      eks.amazonaws.com/role-arn: arn:aws:iam::539702486876:role/myr-EKS_EFS_CSI_DriverRole
```

## and for serviceAccount in the efs-csi-driver-values.yaml

```yaml
# Specifies whether a service account should be created
  serviceAccount:
    create: true
    name: efs-csi-node-sa
    annotations: {
    ## Enable if EKS IAM for SA is used
      eks.amazonaws.com/role-arn: arn:aws:iam::539702486876:role/myr-EKS_EFS_CSI_DriverRole
    }

```

and if you do not have storage class, you can create it

```yaml
storageClasses: {}
  #   name: efs-sc
  #   parameters:
  #    # provisioningMode: efs-ap
  #     provisioner: efs.csi.aws.com
  #     fileSystemId: fs-0069804aebf3a0dce
  #     directoryPerms: "700"
  #   reclaimPolicy: Retain
  #   volumeBindingMode: Immediate
```

```bash
helm show values aws-efs-csi-driver/aws-efs-csi-driver
```

## Install the efs-csi-driver

```bash
helm upgrade --install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --create-namespace \
  -f efs-csi-driver-values.yaml
```

 [text](https://kubernetes-csi.github.io/docs/drivers.html)

## Create policy running file (if you don't have it yet)

```bash
kubectl apply -f efs-service-account.yaml
```

## check if the policy is created

[link](https://oidc.eks.eu-central-1.amazonaws.com/id/31D89A790E11C1822480013FDE7DF626)

## Create the role if( you don't have it yet)

```bash
    aws iam create-role \
    --role-name myr-EKS_EFS_CSI_DriverRole \
    --assume-role-policy-document file://"trust-policy.json"
```

## Attach the policy to the role (if you don't have it yet)

```bash
    aws iam attach-role-policy \
    --policy-arn arn:aws:iam::539702486876:policy/myr-EKS_EFS_CSI_Driver_Policy \
    --role-name myr-EKS_EFS_CSI_DriverRole 
``
## install the efs-csi-driver or update it
```bash
        helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/

        helm repo update aws-efs-csi-driver

        helm upgrade --install aws-efs-csi-driver --namespace kube-system aws-efs-csi-driver/aws-efs-csi-driver \
        --set controller.serviceAccount.create=false \
        --set controller.serviceAccount.name=efs-csi-controller-sa \
        --set useFips=true 
```

## This example shows how to make a static provisioned EFS persistent volume (PV) mounted inside container. Edit Persistent Volume Spec

## Run file `kubectl apply -f man-efs-pv.yaml` to create a Persistent Volume (PV) for EFS

## But first update the `man-efs-pv.yaml` file with the correct EFS ID

## Get the EFS ID

```yaml
aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text
```

## Create a Persistent Volume PVC, PV, and Storage Class

```bash
kubectl apply -f man-efs-pvc.yaml $$ kubectl apply -f man-efs-pv.yaml $$ kubectl apply -f man-efs-sc.yaml
```

## Lets check if the PVC is bounded

```bash
kubectl get pvc
```

## Deploy the application

```bash
kubectl apply -f  man-efs-pod.yaml
```

## Check the status of the pod

```bash
kubectl get pv -w
```

## expected output

```bash
$ kubectl get pv -w
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
efs-pv   5Gi        RWO            Retain           Bound    default/efs-claim                           3m31s
```

## Verify deployment

```bash
kubectl get pods
```

## Also you can verify that data is written onto EFS filesystem

```bash
kubectl exec -ti efs-app -- tail -f /data/out.txt
```
