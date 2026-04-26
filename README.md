# 🐳 Migrate Image Registry to AWS ECR

![AWS](https://img.shields.io/badge/AWS-EKS-orange?logo=amazonaws&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29-blue?logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containers-2496ED?logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-CI/CD-D24939?logo=jenkins&logoColor=white)
![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?logo=terraform&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-Charts-0F1689?logo=helm&logoColor=white)

## 🔍 Key Differences: DockerHub vs Amazon ECR vs Nexus

| Feature              | DockerHub                         | Amazon ECR                          | Nexus Repository                         |
|---------------------|----------------------------------|------------------------------------|------------------------------------------|
| **Type**            | Public/Private SaaS registry     | Managed AWS container registry     | Self-hosted artifact repository          |
| **Authentication**  | Username & password              | `aws ecr get-login-password`        | Username/password or LDAP                |
| **Token Expiry**    | Long-lived                       | 12 hours (auto-refreshed)           | Configurable                             |
| **K8s Integration** | Requires ImagePullSecret         | Native (same AWS account)           | Requires ImagePullSecret                 |
| **Hosting**         | Fully managed (Docker)           | Fully managed (AWS)                 | Self-hosted (VM/K8s)                     |
| **Setup Effort**    | Very low                         | Low (if using AWS)                  | High (install, maintain, scale)          |
| **Cost Model**      | Free tier + paid plans           | Pay per storage (GB) + requests     | Free (OSS) / Paid (Pro license + infra)  |
| **Image Cleanup**   | Manual                           | Lifecycle policies (automated)      | Cleanup policies (configurable)          |
| **Multi-Artifact**  | Containers only                  | Containers only                    | Supports Docker, Maven, npm, etc.        |
| **Best Use Case**   | Simple/public projects           | AWS-native production workloads     | Enterprise artifact management           |

## 📌 What You'll Set Up

- Create a private **Amazon ECR repository**
- Update the **Jenkins pipeline** to build, tag, and push images to ECR
- Configure **EKS to pull images directly from ECR** (same AWS account)

---

## Step 1 — Create an ECR Repository

```bash
aws ecr create-repository \
  --repository-name java-app \
  --region eu-north-1
````

Example response:

```json
{
  "repository": {
    "repositoryUri": "382858226896.dkr.ecr.eu-north-1.amazonaws.com/java-app"
  }
}
```

📝 Save the `repositoryUri` — you will need it in your Jenkinsfile.

---

## Step 2 — Give Jenkins IAM Permissions to Push to ECR

Attach the ECR policy to your IAM user:

```bash
aws iam attach-user-policy \
  --user-name dinho \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess
```

---

## Step 3 — Add AWS Credentials to Jenkins

If not already configured:

**Jenkins UI:**

```
Dashboard → Manage Jenkins → Credentials → System → Global credentials → Add Credentials
```

Add:

| Kind        | ID                    | Value               |
| ----------- | --------------------- | ------------------- |
| Secret text | AWS_ACCESS_KEY_ID     | Your AWS access key |
| Secret text | AWS_SECRET_ACCESS_KEY | Your AWS secret key |

---

## Step 4 — Update Your Jenkinsfile (Use ECR Instead of DockerHub)

```groovy
pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_REGION            = 'eu-north-1'
        CLUSTER_NAME          = 'demo-cluster'
        ECR_REGISTRY          = '382858226896.dkr.ecr.eu-north-1.amazonaws.com'
        ECR_REPO              = 'java-app'
        IMAGE_TAG             = "${BUILD_NUMBER}"
    }

    stages {

        stage('Build Image') {
            steps {
                sh """
                    docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} .
                """
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                    # Authenticate Docker with ECR
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    # Push image to ECR
                    docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
                    # Authenticate with EKS
                    aws eks update-kubeconfig \
                        --name ${CLUSTER_NAME} \
                        --region ${AWS_REGION}

                    # Update deployment with new image
                    kubectl set image deployment/java-app \
                        java-app=${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} \
                        -n fargate-app

                    # Wait for rollout
                    kubectl rollout status deployment/java-app \
                        -n fargate-app \
                        --timeout=300s
                """
            }
        }
    }

    post {
        success {
            echo "✅ Deployed ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} successfully!"
        }
        failure {
            echo "❌ Deployment failed! Rolling back..."
            sh "kubectl rollout undo deployment/java-app -n fargate-app"
        }
    }
}
```

---

## Step 5 — Allow EKS to Pull Images from ECR

Since EKS and ECR are in the same AWS account, attach the read policy to your node group role:

```bash
aws iam attach-role-policy \
  --role-name <NodeGroupRoleName> \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

---

## Step 6 — Commit and Trigger the Pipeline

```bash
git add Jenkinsfile
git commit -m "Switch from DockerHub to ECR"
git push
```

Then monitor Jenkins:

```
Build Image → Push to ECR → Deploy to EKS
```

---

## Step 7 — Verify Images in ECR

```bash
aws ecr list-images \
  --repository-name java-app \
  --region eu-north-1
```

Expected output:

```json
{
  "imageIds": [
    { "imageTag": "1", "imageDigest": "sha256:xxx" },
    { "imageTag": "2", "imageDigest": "sha256:yyy" }
  ]
}
```

---

## ✅ Outcome

* Secure, private container registry on AWS
* Seamless integration with EKS
* Fully automated CI/CD pipeline using Jenkins
* No dependency on external registries (e.g., DockerHub)
