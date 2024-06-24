# Demo How to Setup EMR on EKS with Kyuubi and Karpenter with Steps Explained
# Step -1
Make sure you have aws credentials setup, aws cli, eksctl (https://eksctl.io/installation/), kubectl (1.30, https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/), helm (https://helm.sh/docs/intro/install/) installed.
If you're using cloud9 as your IDE, make sure you have AWS managed temporary credential turned off and remove the `aws_session_token = ` line ~/.aws/credentials.

```
chmod +x ./tools/tool-setup.sh
./tools/tool-setup.sh
```

## Step 0 Create a EKS Cluster with Karpenter installed
Following the steps 1-4 in https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/. 
Recommend to change instance type to m6i.large, CLUSTER_NAME=EMR_EKS
DEFAULT_AZ not necessarily a
```
export AWS_REGION=${AWS_DEFAULT_REGION}
export DEFAULT_AZ=${AWS_REGION}d

aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
```
Following the steps "Create IAM Roles" in https://karpenter.sh/docs/getting-started/migrating-from-cas/

## Step 1 Add Service Accounts to Your EKS Cluster
Create the EMR job execution which has access to your S3 bucket
```
export JOB_EXECUTION_ROLE_NAME=${CLUSTER_NAME}-execution-role
export JOB_EXECUTION_POLICY_NAME=${CLUSTER_NAME}-execution-policy
export JOB_EXECUTION_ROLE_ARN=arn:aws:iam::${AWS_ACCOUNT_ID}:role/${JOB_EXECUTION_ROLE_NAME}

export S3_BUCKET="YOUR_BUCKET"
export EMR_NAMESPACE=emr
<!-- export KYUUBI_NAMESPACE=kyuubi -->
export KYUUBI_SA=emr-kyuubi

chmod +x job-execution-role.sh
./job-execution-role.sh
```
Create the ebs controller service account 
Is this needed? cross account kyuubi execution service account
```
eksctl utils associate-iam-oidc-provider --region=$AWS_DEFAULT_REGION --cluster=$CLUSTER_NAME --approve
envsubst < service-accounts.yaml | eksctl create iamserviceaccount --config-file=- --approve
```
## Step 2 Setup EMR on EKS
```
export EMRCLUSTER_NAME=${CLUSTER_NAME}-emr
chmod +x emr-setup.sh
./emr-setup.sh
```
## Step 3 Create Karpenter and Karpenter NodePool
```
# Logout of helm registry to perform an unauthenticated pull against the public ECR
helm registry logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version "${KARPENTER_VERSION}" --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi 
  
```

创建nodepool和nodeclass
```

envsubst < karpenter-emr-nodepool.yaml | kubectl apply -f -\

```

## Step 4 Test The Vanilla Job Run
Please replace the EMRCLUSTER_ID with your EMR virtual cluster ID
```
sudo yum install jq -y
EMRCLUSTER_ID=$(aws emr-containers list-virtual-clusters | jq -r --arg name "$EMRCLUSTER_NAME" '.virtualClusters[] | select(.name == $name) | .id')

chmod +x start-job-run.sh
./start-job-run.sh
```

## Step 5 Create Pod Templates and Test
In the pod template, we will add node selection to run EMR workloads only on nodes labeled `app=emr` and let 
```
aws s3 cp driver-pod-template.yaml s3://${S3_BUCKET}/pod-template/
aws s3 cp executor-pod-template.yaml s3://${S3_BUCKET}/pod-template/

chmod +x start-job-run-pod-template.sh
./start-job-run-pod-template.sh
```

## Step 6 Create Kyuubi Image and Deploy

Option 1: Build your own Kyuubi image
```
export ECR_URL=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
chmod +x build-kyuubi-docker.sh
./build-kyuubi-docker.sh

envsubst < charts/my-kyuubi-values.yaml | helm install kyuubi charts/kyuubi -n emr --create-namespace -f - --debug
kubectl get pods -n emr
```
Option 2: Use the public image: public.ecr.aws/l2l6g0y5/emr-eks-kyuubi:6.10_180

```
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws
export ECR_URL=public.ecr.aws/l2l6g0y5

```

## Step 7 Login the Kyuubi Pod and Test It
```
kubectl exec -it pod/kyuubi-0 -n emr -- bash
```
```
# execute in the Kyuubi pod's shell. Spark submit test
export S3_BUCKET="YOUR_BUCKET"
aws s3 cp /usr/lib/spark/examples/jars/spark-examples.jar s3://${S3_BUCKET}/jars

/usr/lib/spark/bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--conf spark.executor.instances=5 \
 s3://${S3_BUCKET}/jars/spark-examples.jar 10000
```
```
# beeline test
./bin/beeline -u 'jdbc:hive2://kyuubi-0.kyuubi-headless.emr.svc.cluster.local:10009#spark.app.name=b1;' -n hadoop --hiveconf spark.kubernetes.file.upload.path=s3://${S3_BUCKET}/upload_files/ --hiveconf spark.executor.instances=4 --hiveconf spark.driver.memory=4G --hiveconf spark.driver.cores=4
```