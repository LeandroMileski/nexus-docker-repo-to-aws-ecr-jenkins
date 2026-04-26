*Task - Migrate the existing Kubernetes setup to AWS.*  
*Action:*
1. Provisioned a managed Kubernetes cluster using Amazon EKS  
2. Designed a hybrid compute architecture:   
  EC2 node group for stateful workloads (MySQL)  
  AWS Fargate for serverless application workloads  
3. Re-deployed the application using:  
  Helm charts (MySQL + phpMyAdmin)  
  Kubernetes YAML manifests (Java app)
4. Implemented a CI/CD pipeline with Jenkins:  
  Automated Docker image build & push  
  Enabled rolling deployments using kubectl set image
5. Configured autoscaling:  
  Horizontal Pod Autoscaler (HPA)  
  Cluster Autoscaler for node scaling


This cloud architecture project combines Kubernetes, AWS, CI/CD, and autoscaling.

**Core Components** 
- Infrastructure (AWS EKS)  
  - 1 EKS Cluster (via eksctl)  
    EC2 Node Group (auto-scaled)  
    AWS Fargate profile (serverless workloads)
- Data Layer
  - MySQL deployed via Helm  
    phpMyAdmin for database management
- Application Layer
  - Java microservice deployed on Kubernetes  
    Runs on Fargate (serverless compute)
- CI/CD Pipeline
  - Jenkins pipeline  
    Docker image build & push  
    Automated deployment to EKS
- Scalability & Cost Optimization
  - Horizontal Pod Autoscaler (HPA)
    Cluster Autoscaler (node scaling)  

  
---

# Part 1: Create EKS Cluster with eksctl

**What you'll create:**  
- 1 EKS Cluster  
- 3 Worker Nodes (EC2 node group)  
- 1 Fargate Profile  

---

Step 1 - Create the EKS Cluster with Node Group

(Change region if needed. If you want to use a config file, see: https://docs.aws.amazon.com/eks/latest/eksctl/creating-and-managing-clusters.html)

```bash
eksctl create cluster \
  --name demo-cluster \
  --version 1.29 \
  --region eu-north-1 \
  --nodegroup-name demo-nodes \
  --node-type t3.small \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 3
```

⏳ This takes 15–20 minutes. `eksctl` creates the VPC, subnets, IAM roles, and CloudFormation stacks automatically.

---

Step 2 - Verify the Cluster & Nodes

```bash
eksctl get cluster --name demo-cluster --region eu-north-1
kubectl get nodes
```

---

Step 3 - Create the Fargate Profile

```bash
kubectl create namespace fargate-app

eksctl create fargateprofile \
  --cluster demo-cluster \
  --name demo-fargate-profile \
  --namespace fargate-app \
  --region eu-north-1
```

---

Step 4 - Verify the Fargate Profile

```bash
eksctl get fargateprofile \
  --cluster demo-cluster \
  --region eu-north-1
```

---

Step 5 - Verify Everything Together

```bash
kubectl get nodes
kubectl cluster-info
```

---

# Part 2: Deploy MySQL and phpMyAdmin on EKS

Step 1 - Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

Step 2 - Add Bitnami Repo

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

Step 3 - Create Namespace

```bash
kubectl create namespace mysql-app
```

---

Step 4 - Config (or Clone) MySQL Values File

---

Step 5 - Deploy MySQL

```bash
helm install mysql bitnami/mysql \
  --namespace mysql-app \
  --values mysql-values.yaml
```

---

Step 6 - Config (or clone) phpMyAdmin Values File

---

Step 7 - Deploy phpMyAdmin

```bash
helm install phpmyadmin bitnami/phpmyadmin \
  --namespace mysql-app \
  --values phpmyadmin-values.yaml
```

---

# Part 3: Deploy Java Application on Fargate via YAML file

Step 1 - Config java-app-deployment.yaml

⚠️ Replace <your-dockerhub-or-ecr-image> with your actual image. Fargate requires a valid, accessible container image.

---

Step 2 - Config java-app-service.yaml

---
Step 3 - Deploy application

```bash
kubectl apply -f java-app-deployment.yaml
kubectl apply -f java-app-service.yaml
```
---
Step 4 - Verify Pods are Running on Fargate
```bash
# Watch pods come up (Fargate takes ~1-2 min per pod)
kubectl get pods -n fargate-app -w
```
---
Step 5 - Confirm they're on Fargate nodes:
```bash
kubectl get pods -n fargate-app -o wide
```
The NODE column should show fargate- prefixed node names.

Step 6 - Get the External URL
```bash
kubectl get svc java-app-service -n fargate-app
```
Copy the EXTERNAL-IP and open in your browser:
```http://<EXTERNAL-IP>/```

Step 7 - Verify Replica Count
```bash
kubectl get deployment java-app -n fargate-app
```
Expected:
```
NAME       READY   UP-TO-DATE   AVAILABLE
java-app   3/3     3            3
```
⚠️ Troubleshooting
```
# Pod stuck in Pending?
kubectl describe pod <pod-name> -n fargate-app

# Check Fargate profile is picking up the namespace
eksctl get fargateprofile --cluster demo-cluster --region eu-north-1

# Check logs
kubectl logs <pod-name> -n fargate-app
```

# Part 4: Automate Deployment with Jenkins CD

**What you'll set up:**  
- Jenkins pipeline that builds the Docker image and automatically deploys it to EKS.  
- No more manual kubectl apply after each build.

---

Step 1 — Install kubectl on Jenkins Server
```bash
# SSH into your Jenkins server, then run:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```
---

Step 2 — Install aws-cli on Jenkins Server
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```
---

Step 3 — Add AWS Credentials to Jenkins
In Jenkins UI:
Dashboard → Manage Jenkins → Credentials → System → Global credentials → Add Credentials
Add two credentials:
```
Kind          ID                    Value
Secret text   AWS_ACCESS_KEY_ID     Your AWS access key
Secret text   AWS_SECRET_ACCESS_KEY Your AWS secret key
```

---

Step 4 — Configure kubeconfig on Jenkins Server

Jenkins need to authenticate with your EKS cluster:
```bash
# On the Jenkins server (as jenkins user)
sudo -u jenkins aws configure set aws_access_key_id <YOUR_ACCESS_KEY>
sudo -u jenkins aws configure set aws_secret_access_key <YOUR_SECRET_KEY>
sudo -u jenkins aws configure set region eu-north-1

# Generate kubeconfig for the cluster
sudo -u jenkins aws eks update-kubeconfig \
  --name demo-cluster \
  --region eu-north-1

# Verify Jenkins can reach the cluster
sudo -u jenkins kubectl get nodes
```

---
Step 5 — Update Your Jenkinsfile
Add the deploy stage to your existing pipeline:
```groovy
pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_REGION            = 'eu-north-1'
        CLUSTER_NAME          = 'demo-cluster'
        IMAGE_NAME            = '<your-dockerhub-user>/java-app'
        IMAGE_TAG             = "${BUILD_NUMBER}"
    }

    stages {

        stage('Build Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
                    # Authenticate with EKS
                    aws eks update-kubeconfig \
                        --name ${CLUSTER_NAME} \
                        --region ${AWS_REGION}

                    # Update the deployment image — triggers rolling update
                    kubectl set image deployment/java-app \
                        java-app=${IMAGE_NAME}:${IMAGE_TAG} \
                        -n fargate-app

                    # Wait for rollout to complete
                    kubectl rollout status deployment/java-app \
                        -n fargate-app \
                        --timeout=300s
                """
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful! Image: ${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo "❌ Deployment failed! Rolling back..."
            sh "kubectl rollout undo deployment/java-app -n fargate-app"
        }
    }
}
```
---

Step 6 — Commit the Jenkinsfile
```bash
git add Jenkinsfile
git commit -m "Add CD: auto deploy to EKS after build"
git push
```

---

Step 7 — Trigger and Verify the Pipeline  
In Jenkins, trigger a build and watch the stages:  
Build Image → Push Image → Deploy to EKS

Then verify the rollout in your cluster:
```bash
# Check new pods are running
kubectl get pods -n fargate-app

# Confirm the new image is being used
kubectl describe deployment java-app -n fargate-app | grep Image

# Check rollout history
kubectl rollout history deployment/java-app -n fargate-app
```

How it works end-to-end:
```
Developer pushes code
        ↓
Jenkins detects change (webhook/poll)
        ↓
Builds new Docker image tagged with BUILD_NUMBER
        ↓
Pushes image to DockerHub
        ↓
kubectl set image → triggers rolling update on EKS
        ↓
Fargate spins up new pods with new image
        ↓
Old pods terminated after new ones are healthy
        ↓
Rollback automatically if deployment fails
```

---

# Part 5: Configure Autoscaling on EKS
**What you'll set up**  
- Cluster Autoscaler — scales EC2 nodes in/out (1 min, 3 max)
- HPA (Horizontal Pod Autoscaler) — scales pods up/down based on CPU usage
- Together they ensure you only pay for what you use

---

How it works together
```
Low traffic → HPA scales pods down → Cluster Autoscaler removes underutilized nodes → cost savings
High traffic → HPA scales pods up → Cluster Autoscaler adds nodes → handles the load
```

---

Step 1 — Tag your Node Group for Cluster Autoscaler
The Cluster Autoscaler needs specific tags on your Auto Scaling Group to discover it:
```bash
# Get your Auto Scaling Group name
aws autoscaling describe-auto-scaling-groups \
  --region eu-north-1 \
  --query "AutoScalingGroups[?contains(Tags[?Key=='eks:cluster-name'].Value, 'demo-cluster')].AutoScalingGroupName" \
  --output text
```

Then add the required tags:
```bash
aws autoscaling create-or-update-tags \
  --region eu-north-1 \
  --tags \
    ResourceId=<ASG_NAME>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=true \
    ResourceId=<ASG_NAME>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/demo-cluster,Value=owned,PropagateAtLaunch=true
```

---

Step 2 — Create IAM Policy for Cluster Autoscaler  
Using `cluster-autoscaler-policy.json`:
```bash
aws iam create-policy \
  --policy-name ClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json
```
Attach it to your node group role:
```bash
aws iam attach-role-policy \
  --role-name <NodeGroupRoleName> \
  --policy-arn arn:aws:iam::382858226896:policy/ClusterAutoscalerPolicy
```
---
Step 3 — Deploy the Cluster Autoscaler
Using `cluster-autoscaler.yaml`
```bash
kubectl apply -f cluster-autoscaler.yaml
```
---
Step 4 — Verify Cluster Autoscaler is Running
```bash
kubectl get pods -n kube-system | grep cluster-autoscaler

# Check logs to confirm it detected your node group
kubectl logs -n kube-system \
  -l app=cluster-autoscaler \
  --tail=20
```

You should see something like:
```
Found node group: eksctl-demo-cluster-nodegroup-demo-nodes-NodeGroup
Min: 1, Max: 3, Current: 3
```
---
Step 5 — Install Metrics Server (required for HPA)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify it's running
kubectl get deployment metrics-server -n kube-system
```
---
Step 6 — Configure HPA for your Java App
```bash
kubectl autoscale deployment java-app \
  --namespace fargate-app \
  --cpu-percent=50 \
  --min=1 \
  --max=3
```
Or use YAML file `java-app-hpa.yaml`.
```bash
kubectl apply -f java-app-hpa.yaml
```
---
Step 7 — Verify Autoscaling is Configured
```bash
# Check HPA status
kubectl get hpa -n fargate-app

# Check current node count
kubectl get nodes

# Watch HPA in real time
kubectl get hpa -n fargate-app -w
```
Expected HPA output:
```
NAME           REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS
java-app-hpa   Deployment/java-app   5%/50%    1         3         3
```

Summary:
```
Component           Min     Max       Scales based on  
Cluster Autoscaler  1 node  3 nodes   Unschedulable pods / underutilized nodes  
HPA                 1 pod   3 pods    CPU utilization (threshold: 50%)  
```
```
Scenario                What happens  
Weekend / low traffic   HPA scales pods to 1 → Autoscaler removes idle nodes → 1 node running  
Weekday / high traffic  HPA scales pods to 3 → Autoscaler adds nodes if needed → up to 3 nodes
```
