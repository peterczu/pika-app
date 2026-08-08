# pika-app 🚀

A production-grade containerized Nginx application deployed on AWS ECS Fargate with a fully automated CI/CD pipeline via GitHub Actions.

---

## 🏗️ Architecture

```
Internet
    ↓
ALB (public subnets)
    ↓
ECS Fargate (private subnets)
    ↓
ECR via VPC Endpoint
    ↓
CloudWatch Logs via VPC Endpoint
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Container Runtime | AWS ECS Fargate |
| Container Registry | AWS ECR |
| Load Balancer | AWS ALB |
| Networking | AWS VPC — public & private subnets |
| CI/CD | GitHub Actions (OIDC) |
| Infrastructure | Terraform |
| App | Nginx |

---

## 🔐 Security Wins

- **Fargate on private subnets** — containers have no public IP, never directly exposed to internet
- **VPC Endpoints** — ECR image pulls and CloudWatch logs traffic never leaves the AWS network
- **OIDC authentication** — no long lived AWS credentials stored anywhere
- **Immutable image tags** — images tagged with git commit SHA, no overwriting
- **ALB as single entry point** — only controlled ingress into private network
- **No SSH access** — fully managed serverless containers, no need to touch servers

---

## ⚙️ CI/CD Pipeline

Every push to `main` triggers:

```
git push
    ↓
GitHub Actions (OIDC — no credentials stored)
    ↓
Docker image built
    ↓
Image pushed to ECR (tagged with commit SHA)
    ↓
ECS task definition updated with new image
    ↓
ECS Fargate deploys new task
    ↓
ALB routes traffic to new container
    ↓
Live ✅
```

---

## 👨‍💻 Author

**Peter Madueke**
- GitHub: [@peterczu](https://github.com/peterczu)
- Email: ipeterdevops@gmail.com
- LinkedIn: [peter-madueke](https://linkedin.com/in/peter-madueke)
