```
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup 

```
2. step
   ```
   eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve

   ```
   3. step
      ```
      eksctl create nodegroup --cluster=eksdemo1 \
                        --region=us-east-1 \
                        --name=eksdemo1-ng-private1 \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking    
      ```
     ### Pre-requisite-3:  Verify Cluster, Node Groups and configure kubectl cli if not configured
1. EKS Cluster
2. EKS Node Groups in Private Subnets

# Verfy EKS Cluster
```
eksctl get cluster
```
# Verify EKS Node Groups
```
eksctl get nodegroup --cluster=eksdemo1
```
# Delete Cluster
```
eksctl delete cluster eksdemo1
```
# Verify if any IAM Service Accounts present in EKS Cluster
```
eksctl get iamserviceaccount --cluster=eksdemo1
```
Observation:
1. No k8s Service accounts as of now. 

# Configure kubeconfig for kubectl
eksctl get cluster # TO GET CLUSTER NAME
aws eks --region <region-code> update-kubeconfig --name <cluster_name>
```
aws eks --region us-east-1 update-kubeconfig --name eksdemo1
```

# Verify EKS Nodes in EKS Cluster using kubectl
```
kubectl get nodes
```



# Download IAM Policy
## Download latest
```
curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```


# Create IAM Policy using policy downloaded 
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_latest.json
```    

- **Important Note:** If you view the policy in the AWS Management Console, you may see warnings for ELB. These can be safely ignored because some of the actions only exist for ELB v2. You do not see warnings for ELB v2.

### Make a note of Policy ARN    
- Make a note of Policy ARN as we are going to use that in next step when creating IAM Role.

# Policy ARN 
Policy ARN:  arn:aws:iam::180789647333:policy/AWSLoadBalancerControllerIAMPolicy


## Step-03: Create an IAM role for the AWS LoadBalancer Controller and attach the role to the Kubernetes service account 
- Applicable only with `eksctl` managed clusters
- This command will create an AWS IAM role 
- This command also will create Kubernetes Service Account in k8s cluster
- In addition, this command will bound IAM Role created and the Kubernetes service account created
### Step-03-01: Create IAM Role using eksctl

# Verify if any existing service account
```
kubectl get sa -n kube-system
```
```
kubectl get sa aws-load-balancer-controller -n kube-system
```
Obseravation:
1. Nothing with name "aws-load-balancer-controller" should exist

# Replaced name, cluster and policy arn (Policy arn we took note in step-02)
```
eksctl create iamserviceaccount \
  --cluster=eksdemo1 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::180789647333:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

# Get IAM Service Account
```
eksctl  get iamserviceaccount --cluster eksdemo1
```
Verify k8s Service Account using kubectl
```
kubectl get sa -n kube-system
```
```
kubectl get sa aws-load-balancer-controller -n kube-system
```
Obseravation:
1. We should see a new Service account created. 


## Step-04: Install the AWS Load Balancer Controller using Helm V3 
### Step-04-01: Install Helm
- [Install Helm](https://helm.sh/docs/intro/install/) if not installed
- [Install Helm for AWS EKS](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)

# Install Helm (if not installed) Windows
```
choco install kubernetes-helm
```

# Verify Helm version
```
helm version
```
### Step-04-02: Install AWS Load Balancer Controller
- **Important-Note-1:** If you're deploying the controller to Amazon EC2 nodes that have restricted access to the Amazon EC2 instance metadata service (IMDS), or if you're deploying to Fargate, then add the following flags to the command that you run:
```
--set region=region-code
--set vpcId=vpc-xxxxxxxx
```
- **Important-Note-2:** If you're deploying to any Region other than us-west-2, then add the following flag to the command that you run, replacing account and region-code with the values for your region listed in Amazon EKS add-on container image addresses.
- [Get Region Code and Account info](https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html)

--set image.repository=account.dkr.ecr.region-code.amazonaws.com/amazon/aws-load-balancer-controller

# Add the eks-charts repository.
```
helm repo add eks https://aws.github.io/eks-charts
```
# Update your local repo to make sure that you have the most recent charts.
```
helm repo update
```

## Replace Cluster Name, Region Code, VPC ID, Image Repo Account ID and Region Code  
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eksdemo1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-016b2f6ed3d4fc306 \
  --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller
```
### Step-04-03: Verify that the controller is installed and Webhook Service created

# Verify that the controller is installed.
```
kubectl -n kube-system get deployment 
```
kubectl -n kube-system get deployment aws-load-balancer-controller
```
kubectl -n kube-system describe deployment aws-load-balancer-controller
```
# Sample Output
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           27s
```

# Uninstall AWS Load Balancer Controller
```
helm uninstall aws-load-balancer-controller -n kube-system 
```
# Create Ingress resource 
```
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: my-aws-ingress-class
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb

```
# Create IngressClass Resource
```
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: my-aws-ingress-class
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb
```
# Verify IngressClass Resource
```
kubectl get ingressclass
```
# Describe IngressClass Resource
```
kubectl describe ingressclass my-aws-ingress-class
```
# ALB ingress Yml

# Annotations Reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginxapp1
  labels:
    app: app1-nginx
  annotations:
    # Load Balancer Name
    alb.ingress.kubernetes.io/load-balancer-name: app1ingress
    #kubernetes.io/ingress.class: "alb" (OLD INGRESS CLASS NOTATION - STILL WORKS BUT RECOMMENDED TO USE IngressClass Resource) # Additional Notes: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/ingress_class/#deprecated-kubernetesioingressclass-annotation
    # Ingress Core Settings
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Health Check Settings
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP 
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    alb.ingress.kubernetes.io/healthcheck-path: /app1/index.html    
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/success-codes: '200'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
spec:
  ingressClassName: my-aws-ingress-class # Ingress Class
  defaultBackend:
    service:
      name: app1-nginx-nodeport-service
      port:
        number: 80                  
 ```     

# 1. If  "spec.ingressClassName: my-aws-ingress-class" not specified, will reference default ingress class on this kubernetes cluster
# 2. Default Ingress class is nothing but for which ingress class we have the annotation `ingressclass.kubernetes.io/is-default-class: "true"`





---
title: AWS Load Balancer Controller - External DNS Install
description: Learn AWS Load Balancer Controller - External DNS Install
---

## Step-01: Introduction
- **External DNS:** Used for Updating Route53 RecordSets from Kubernetes 
- We need to create IAM Policy, k8s Service Account & IAM Role and associate them together for external-dns pod to add or remove entries in AWS Route53 Hosted Zones. 
- Update External-DNS default manifest to support our needs
- Deploy & Verify logs

## Step-02: Create IAM Policy
- This IAM policy will allow external-dns pod to add, remove DNS entries (Record Sets in a Hosted Zone) in AWS Route53 service
- Go to Services -> IAM -> Policies -> Create Policy
  - Click on **JSON** Tab and copy paste below JSON
  - Click on **Visual editor** tab to validate
  - Click on **Review Policy**
  - **Name:** AllowExternalDNSUpdates 
  - **Description:** Allow access to Route53 Resources for ExternalDNS
  - Click on **Create Policy**  

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```
- Make a note of Policy ARN which we will use in next step
```t
# Policy ARN
arn:aws:iam::180789647333:policy/AllowExternalDNSUpdates
```  


## Step-03: Create IAM Role, k8s Service Account & Associate IAM Policy
- As part of this step, we are going to create a k8s Service Account named `external-dns` and also a AWS IAM role and associate them by annotating role ARN in Service Account.
- In addition, we are also going to associate the AWS IAM Policy `AllowExternalDNSUpdates` to the newly created AWS IAM Role.
### Step-03-01: Create IAM Role, k8s Service Account & Associate IAM Policy
```t
# Template
eksctl create iamserviceaccount \
    --name service_account_name \
    --namespace service_account_namespace \
    --cluster cluster_name \
    --attach-policy-arn IAM_policy_ARN \
    --approve \
    --override-existing-serviceaccounts

# Replaced name, namespace, cluster, IAM Policy arn 
eksctl create iamserviceaccount \
    --name external-dns \
    --namespace default \
    --cluster eksdemo1 \
    --attach-policy-arn arn:aws:iam::180789647333:policy/AllowExternalDNSUpdates \
    --approve \
    --override-existing-serviceaccounts
```
### Step-03-02: Verify the Service Account
- Verify external-dns service account, primarily verify annotation related to IAM Role
```t
# List Service Account
kubectl get sa external-dns

# Describe Service Account
kubectl describe sa external-dns
Observation: 
1. Verify the Annotations and you should see the IAM Role is present on the Service Account
```
### Step-03-03: Verify CloudFormation Stack
- Go to Services -> CloudFormation
- Verify the latest CFN Stack created.
- Click on **Resources** tab
- Click on link  in **Physical ID** field which will take us to **IAM Role** directly

### Step-03-04: Verify IAM Role & IAM Policy
- With above step in CFN, we will be landed in IAM Role created for external-dns. 
- Verify in **Permissions** tab we have a policy named **AllowExternalDNSUpdates**
- Now make a note of that Role ARN, this we need to update in External-DNS k8s manifest
```t
# Make a note of Role ARN
arn:aws:iam::180789647333:role/eksctl-eksdemo1-addon-iamserviceaccount-defa-Role1-JTO29BVZMA2N
```

### Step-03-05: Verify IAM Service Accounts using eksctl
- You can also make a note of External DNS Role ARN from here too. 
```t
# List IAM Service Accounts using eksctl
eksctl get iamserviceaccount --cluster eksdemo1

# Sample Output
Kalyans-Mac-mini:08-06-ALB-Ingress-ExternalDNS kalyanreddy$ eksctl get iamserviceaccount --cluster eksdemo1
2022-02-11 09:34:39 [ℹ]  eksctl version 0.71.0
2022-02-11 09:34:39 [ℹ]  using region us-east-1
NAMESPACE	NAME				ROLE ARN
default		external-dns			arn:aws:iam::180789647333:role/eksctl-eksdemo1-addon-iamserviceaccount-defa-Role1-JTO29BVZMA2N
kube-system	aws-load-balancer-controller	arn:aws:iam::180789647333:role/eksctl-eksdemo1-addon-iamserviceaccount-kube-Role1-EFQB4C26EALH
Kalyans-Mac-mini:08-06-ALB-Ingress-ExternalDNS kalyanreddy$ 
```


## Step-04: Update External DNS Kubernetes manifest
- **Original Template** you can find in https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md
- **File Location:** kube-manifests/01-Deploy-ExternalDNS.yml
### Change-1: Line number 9: IAM Role update
  - Copy the role-arn you have made a note at the end of step-03 and replace at line no 9.
```yaml
    eks.amazonaws.com/role-arn: arn:aws:iam::180789647333:role/eksctl-eksdemo1-addon-iamserviceaccount-defa-Role1-JTO29BVZMA2N
```
### Chnage-2: Line 55, 56: Commented them
- We used eksctl to create IAM role and attached the `AllowExternalDNSUpdates` policy
- We didnt use KIAM or Kube2IAM so we don't need these two lines, so commented
```yaml
      #annotations:  
        #iam.amazonaws.com/role: arn:aws:iam::ACCOUNT-ID:role/IAM-SERVICE-ROLE-NAME    
```
### Change-3: Line 65, 67: Commented them
```yaml
        # - --domain-filter=external-dns-test.my-org.com # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
       # - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
```

### Change-4: Line 61: Get latest Docker Image name
- [Get latest external dns image name](https://github.com/kubernetes-sigs/external-dns/releases/tag/v0.10.2)
```yaml
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.10.2
```

## Step-05: Deploy ExternalDNS
#get iam role arn 
eksctl get iamserviceaccount --cluster eksdemo1
```
PS D:\Devops\k8s master class> eksctl get iamserviceaccount --cluster eksdemo1
NAMESPACE       NAME                            ROLE ARN
default         external-dns                    arn:aws:iam::617729733308:role/eksctl-eksdemo1-addon-iamserviceaccount-defa-Role1-ZZWBH9CCSS50
kube-system     aws-load-balancer-controller    arn:aws:iam::617729733308:role/eksctl-eksdemo1-addon-iamserviceaccount-kube-Role1-MECFOQ66OZ2T
PS D:\Devops\k8s master class> 
```
copy  external-dns    arn  arn:aws:iam::617729733308:role/eksctl-eksdemo1-addon-iamserviceaccount-defa-Role1-ZZWBH9CCSS50
and past in Deploy-ExternalDNS.yml line no 9
01-Deploy-ExternalDNS.yml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  # If you're using Amazon EKS with IAM Roles for Service Accounts, specify the following annotation.
  # Otherwise, you may safely omit it.
  annotations:
    # Substitute your account ID and IAM service role name below. #Change-1: Replace with your IAM ARN Role for extern-dns
    eks.amazonaws.com/role-arn: arn:aws:iam::617729733308:role/eksctl-omrs-addon-iamserviceaccount-default-Role1-1K5PNMC496DFH
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
      # If you're using kiam or kube2iam, specify the following annotation.
      # Otherwise, you may safely omit it.  #Change-2: Commented line 55 and 56
      #annotations:  
        #iam.amazonaws.com/role: arn:aws:iam::ACCOUNT-ID:role/IAM-SERVICE-ROLE-NAME    
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.10.2
        args:
        - --source=service
        - --source=ingress
        # Change-3: Commented line 65 and 67 - --domain-filter=external-dns-test.my-org.com # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
        - --provider=aws
       # Change-3: Commented line 65 and 67  - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
        - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
        - --registry=txt
        - --txt-owner-id=my-hostedzone-identifier
      securityContext:
        fsGroup: 65534 # For ExternalDNS to be able to read Kubernetes and AWS token files

```
- Deploy the manifest
```t
# Change Directory
cd 08-06-Deploy-ExternalDNS-on-EKS

# Deploy external DNS
kubectl apply -f kube-manifests/

# List All resources from default Namespace
kubectl get all

# List pods (external-dns pod should be in running state)
kubectl get pods

# Verify Deployment by checking logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

## References
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/alb-ingress.md
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md


