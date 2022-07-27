## EFS Setup for EKS cluster

<br/>

### Pre-Requisites
***
<br/>
1. AWS (IAM) OpenID Connect (OIDC) provider for your cluster.

  > To determine whether you already have one, or to create one follow this [OIDC Provider setup](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)

<br/>

2. _**aws-cli**_ with version **2.7.13** to greater installed.

<br/>

3. _**kubectl**_ utility is installed and configured with the EKS cluster.

<br/>
<br/>

> ###  Create an IAM Policy and Role 

<br/>

**Create an IAM Policy and attach it to an IAM Role inorder to allow the EFS Driver to interact with our Filesystem.**

<br/>

>  **Steps to Deploy the EFS CSI Driver to the EKS Cluster.**

<br/>

1.  Download the IAM Policy document.
<br/>

```Bash 
curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json
```

<br/>

2. Create the policy also you can change the name of _**AmazonEKS_EFS_CSI_Driver_Policy**_ , However make sure to change the name in the later steps too accordingly.
<br/>

```Bash
aws iam create-policy \
    --policy-name **AmazonEKS_EFS_CSI_Driver_Policy** \
    --policy-document file://iam-policy-example.json
```
<br/>

3. Create an IAM role and attach the IAM policy to it. Annotate the Kubernetes service account with the IAM role ARN and the IAM role with the Kubernetes service account name.
  * Replace _**my-cluster**_ with your cluster name and _**111122223333**_ with your account ID. Replace _**region-code**_ with the AWS Region that your cluster is in.
<br/>

```Bash
eksctl create iamserviceaccount \
    --cluster **my-cluster** \
    --namespace kube-system \
    --name efs-csi-controller-sa \
    --attach-policy-arn arn:aws:iam::**111122223333**:policy/AmazonEKS_EFS_CSI_Driver_Policy \
    --approve \
    --region **region-code**
```

<br/>
<br/>

>  **Install the EFS Driver**

<br/>

**This procedure requires Helm V3 with appropriate version compatible with the EKS version.**

<br/>

1. Add the helm repo.

<br/>

```bash
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
```
2. Update the repo.

<br/>

```bash
helm repo update
```

3. Install a release of the driver using the Helm chart. Replace the Repository Address [Using this link for container images](https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html)

<br/>

```bash
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=**602401143452_.dkr.ecr._region-code_.**amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa
```
<br/>

>  **Create an EFS File System**

<br/>

**Creating EFS FileSystem for EKS Cluster.**

<br>

1. Retrieve the VPC ID that your cluster is in and store it in a variable for use in a later step. Replace _**my-cluster**_ with your cluster name.

<br/>

```bash
vpc_id=$(aws eks describe-cluster \
    --name my-cluster \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)
```

<br/>

2. Retrieve the CIDR range for your cluster's VPC and store it in a variable for use in a later step.

<br/>

```bash
cidr_range=$(aws ec2 describe-vpcs \
    --vpc-ids $vpc_id \
    --query "Vpcs[].CidrBlock" \
    --output text)
```

<br/>

3. Create a security group with an inbound rule that allows inbound NFS traffic for your Amazon EFS mount points.
 * You can replace the example _**values**_ with your own.

<br/>

```bash
security_group_id=$(aws ec2 create-security-group \
    --group-name **MyEfsSecurityGroup** \
    --description "My EFS security group" \
    --vpc-id $vpc_id \
    --output text)
```

<br>

 * Create an inbound rule that allows inbound NFS traffic from the CIDR for your cluster's VPC.

<br/>

```bash
aws ec2 authorize-security-group-ingress \
    --group-id $security_group_id \
    --protocol tcp \
    --port 2049 \
    --cidr $cidr_range
```

<br/>

4. Create an Amazon EFS file system for your Amazon EKS cluster. 
 * Replace the _**region-code**_ with your clusters region-code.

<br/>

```bash
file_system_id=$(aws efs create-file-system \
    --region **region-code** \
    --performance-mode generalPurpose \
    --query 'FileSystemId' \
    --output text)
```

<br/>

 * Create Mount Targets

<br/>

 i) Determine the IP addresses of CLuster nodes 
  
 <br/>

 ```bash
 kubectl get nodes
 ```

<br>

ii) Determine the IDs of the subnets in your VPC and which Availability Zone the subnet is in.

<br/>

```bash
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$vpc_id" \
    --query 'Subnets[*].{SubnetId: SubnetId,AvailabilityZone: AvailabilityZone,CidrBlock: CidrBlock}' \
    --output table
```

<br/>

iii) Add mount targets for the subnets that your nodes are in, This should be run multiple times for all the subnets that our nodes are in. Replacing _**subnet-EXAMPLEe2ba886490**_ with the appropriate subnet ID.

<br>

```bash
aws efs create-mount-target \
    --file-system-id $file_system_id \
    --subnet-id **subnet-EXAMPLEe2ba886490** \
    --security-groups $security_group_id
```

<br/>

5. Test the EFS setup by following the **Deploy a Sample Application** section in [This link](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)

