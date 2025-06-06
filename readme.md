# 5 Microservice application on AWS EKS with aws native setup

![architecture-diagram](./setup.png)

- A Multi-Service voting application with real-time results, built with:

  - Frontend: Python/Flask
  - Worker: .NET Core
  - Results: Node.js/Express
  - Data: Redis + PostgreSQL

## 🌟 Key Features
- **AWS Infrastructure**  
  - **AWS ECR** for a secure private container registory 
  - **ALB Ingress Controller** with IP-mode routing  
  - **TLS Termination** using AWS ACM certificates  
  - **ExternalDNS** for automatic Route53 record management  
  - **EBS CSI Driver** for dynamic PostgreSQL volume provisioning  
  - **AWS Secrets Manager** for storing kubenetes secrets securly
  - **Pod Identity** for secure AWS resource access  

---
### Prerequisites
- AWS Account with EKS cluster (1.24+) configured with pod identity addon
- `kubectl`, `awscli`, and `helm` configured
- Route53 hosted zone with a Domain name preconfigured with name-servers

# DEPLOYMENT
**make sure the pod identity is already setup**
> To use pod identity, look for it in addons and  enable it from console 

**create the namespace**
`kubectl apply -f vote-namespace.yaml`

**apply the configmap**
`kubectl apply -f vote-configmap.yaml`

**create the redis deployment with its service**
`kubectl apply -f vote-redis.yaml`
---
## ECR Private repository access using pod identity assosiation
- create an iam role with permission policy `iam_ECR.json` and bind them together
- create a service account using `ecr_SA.yaml`
- set up pod identity assosiation 
    ```bash
  aws eks create-pod-identity-association \
  --cluster-name YOUR_CLUSTER \
  --namespace vote \
  --service-account ecr-vote-access-sa \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/YOUR_ECR_ROLE
    ```
- now we just have to mention our image uri without any credentials just a serviceaccount tag 



## Secrets setup using Secrets store csi driver to access secrets from AWS SECRETS MANAGER

### setup pod identity for authentication and access of secrets form aws secrets manager
- create an iam policy to allow access to aws secrets store. use the `iam_secretsAWS.json` for policy
- create an iam role for EKS service -> pod identity and attach the previously created role to this account
- create a service account using the `aws_sec_SA.yaml` 
- enable pod idnetity 
    ```bash
      aws eks create-pod-identity-association \
        --cluster-name YOUR_CLUSTER_NAME \
        --namespace vote \
        --service-account vote-secrets-sa \
        --role-arn arn:aws:iam::YOUR_ACCOUNT_ID:role/EKSSecretsAccessRole
    ```
#### install secrets store csi driver and provider
- install the secrets store driver
- this will be done it 2 parts where we install the driver and aws provider
    - secrets-store-csi-driver provides the native k8s implementaiton 
    - aws provider translates aws secret requests to secret manger api calls 

  ```bash
  helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
  helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system \
  --set syncSecret.enabled=true \
  --set enableSecretRotation=true
  ```
- install the aws provider
    ```bash
        helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
        helm install -n kube-system aws-provider aws-secrets-manager/secrets-store-csi-driver-provider-aws
    ```
- verify the installation
    ```bash
      kubectl get pods -n kube-system -l "app in (secrets-store-csi-driver, secrets-store-csi-driver-provider-aws)"
    ```
**now we have to create secretproviderclass for each application that needs secrets in the same ns as application**
- we can either use volume mounts or k8s native secrets approach where it will create a default secrets in cluster 
- `kubectl apply -f vote_secretclass.yaml`
- it will fetch the secrets, and create a k8s native secrect which we can use to apply secrets as env variables in pods instead of volume mounts 


## postgres deployment using AWS EBS volumes

### setup EBS CSI for Persistant Volume for PostgresDB
1. create an IAM Role for eks pod identity with policy `AmazonEBSCSIDriverPolicy` attached
2. create EBS service account `kubectl apply -f ebs_SA.yaml`
3. create pod identity association 
      ```bash
          aws eks create-pod-identity-association \
          --cluster-name YOUR_CLUSTER_NAME \
          --namespace kube-system \
          --service-account ebs-csi-controller-sa \ 
          --role-arn arn:aws:iam::ACCOUNT_ID:role/EBS_CSI_Driver_Role # arn of iam role created with ebscsidriverpolicy
      ```
4. deploy EBS CSI driver using the `ebs-csi-controller-sa` service account we just created
      ```bash
        helm repo add eks https://aws.github.io/eks-charts
        helm upgrade --install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
        --namespace kube-system \
        --set controller.serviceAccount.create=false \  # since we already have sa configured
        --set controller.serviceAccount.name=ebs-csi-controller-sa \  # Match SA name
        --set podIdentity.enabled=true \
        --set podIdentity.assumeRoleArn=arn:aws:iam::ACCOUNT_ID:role/EBS_CSI_Driver_Role
      ```
5. verify
    ```bash
    kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
    kubectl logs -n kube-system deployment/ebs-csi-controller -c csi-provisioner
    ```
**now we can deploy our postgresdb**

**create storage class for postgres using EBS volume for storage**
| EBS only allows `ReadWriteOnce` with volumes, EFS allows multipod readwriteaccess but costs more than EBS

**deploy the postgresdb**
`kubectl apply -f ebs_vote_SC.yaml`
`kubectl apply -f vote_pvc_pg.yaml`
`kubectl apply -f voting_postgres.yaml`

**our databases are ready (redis and postgress)**
**deploy the application**
`kubectl apply -f voting-worker.yaml`
`kubectl apply -f voting-vote.yaml`
`kubectl apply -f voting-result.yaml`
---

### SETUP ExternalDNS for automatic Route53 DNS record update
- when we create ingress using aws alb, we get a dns record for our application 
- we do not give this dns record to end user, insted we create a records to route our traffic to our domain name
- we can manually do this everytime, but we can automate this using aws ExternalDNS addons
- we will specify an annotation, and when we will create our ingress deployment, it will automatically create A,CNAM,TXT,AAAA records for our application
- lets do this

**setup externalDNS**
- create a custom IAM policy using the provided policy json and create a role for eks pod identity and attch that policy to this role
- create a service account for externalDNS `kubectl apply -f dns_SA.yaml`
- link role to service account via pod identity
    ```bash
    aws eks create-pod-identity-association \
      --cluster-name YOUR_CLUSTER_NAME \
      --namespace kube-system \
      --service-account external-dns \
      --role-arn arn:aws:iam::ACCOUNT_ID:role/ExternalDNSRole
    ```
- install externalDNS via helm
  ```bash
      helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
      helm upgrade --install external-dns external-dns/external-dns \
        --namespace kube-system \
        --set provider=aws \
        --set aws.zoneType=public \
        --set domainFilters[0]=homelain.click \
        --set policy=sync \
        --set serviceAccount.create=false \
        --set serviceAccount.name=external-dns \
        --set podIdentity.enabled=true \
        --set podIdentity.assumeRoleArn=arn:aws:iam::ACCOUNT_ID:role/ExternalDNSRole
  ```
---
### Setup ingress With SSL using AWS ALB controller
- when using AWS ALB controller for ingress, we dont need to setup any certmanager in our cluster since SSL Termination is happening at LoadBalancer level and loadbalancer forwards the request in HTTP.
- AWS ALB forwards the traffic directly to POD since aws VPC CNI is used in eks so PODS recieve IP from VPC CIDR.
- ALB controller create target group of pod ips and monitors the cluster for any new pod 
- ALB controller automatically updates the target group to latest pod ip

- lets set it up

1. create a seperate IAM role with policy `AWSLoadBalancerControllerIAMPolicy` attached 
2. create a service account for ALB `kubectl apply -f alb_SA.yaml`
3. link role to service account via pod identity
      ```bash
        helm repo add eks https://aws.github.io/eks-charts
        aws eks create-pod-identity-association \
        --cluster-name YOUR_CLUSTER \
        --namespace kube-system \
        --service-account aws-load-balancer-controller \
        --role-arn arn:aws:iam::ACCOUNT_ID:role/AWSLoadBalancerControllerRole
      ```
4. install ALB controller via Helm
      ```bash
        helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
          --namespace kube-system \
          --set clusterName=YOUR_CLUSTER \
          --set serviceAccount.create=false \
          --set serviceAccount.name=aws-load-balancer-controller \
          --set podIdentity.enabled=true \
          --set podIdentity.assumeRoleArn=arn:aws:iam::ACCOUNT_ID:role/AWSLoadBalancerControllerRole
      ```
5. verify the installation
    ```bash
    kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
    kubectl logs -n kube-system deployment/aws-load-balancer-controller | grep "successfully acquired lease"
    ```

6. to make sure load balancer is getting created in correct subnet we can tag our desired subnet like this, which later ingress annotations will use
  - public subnets for internet facing albs, repeat for all public subnets needed
    ```bash
      aws ec2 create-tags \
      --resources subnet-12345678 \
      --tags Key=kubernetes.io/role/elb,Value=1
    ```
  - private subnets for internal albs
      ```bash
      aws ec2 create-tags \
      --resources subnet-12345678 \
      --tags Key=kubernetes.io/role/internal-elb,Value=1
      ```

| in order to use SSL we have to use AWS ACM to get a ssl certificate for our domain

7. go to aws acm -> request certificate -> public certificate -> enter all the subdomains -> request -> import to route53

**lets apply the ingress**
- in this ingress file, we are using arn of ssl certificate for ssl support
- target-type ip to forward traffic from alb to pod directly 
- redirecting http to https
- listener ports for alb to define which ports to listen traffic 

**apply ingress file**
`kubectl apply -f vote_ingress.yaml`
`kubectl get ingress ` to see the dns created by loadbalancer


#### now simpley wait for loadbalancer and dns records to provision successfully and we can access our application through our hostname with amazon certified ssl certs



# FURTHER IMPROVEMENTS TO-DO
- [x] add resource limit for applications
- [ ] add liveness probes 
- [x] migrate from dockerhub to ECR
- [x] add aws secret manager controller for secrets