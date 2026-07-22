# Coworking Space Service — Analytics API Deployment

## Overview
This project deploys the Coworking Space **Analytics API** to a Kubernetes cluster running on **AWS EKS**, with images built and published through a **CodeBuild → ECR** pipeline. The application is a Python (Flask) microservice that reads user check-in data from a PostgreSQL database and exposes reporting endpoints.

## Technologies
- **Docker** packages the Python app into a portable image.
- **AWS CodeBuild** builds the image and pushes it to **Amazon ECR** using semantic version tags (`1.0.<build-number>`).
- **Amazon EKS + kubectl** run the workload in Kubernetes.
- **ConfigMap** holds non-sensitive settings (`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USERNAME`); a **Secret** holds `DB_PASSWORD`.
- **AWS CloudWatch Container Insights** collects pod logs and health output.

## Architecture / Deploy Flow
A commit to the repository triggers CodeBuild, which authenticates to ECR, builds the Docker image from `analytics/Dockerfile`, tags it with the build number, and pushes it to ECR. Kubernetes then pulls that image for the `coworking` Deployment, which is fronted by a `LoadBalancer` Service on port `5153`. PostgreSQL runs in-cluster with a PersistentVolumeClaim so data survives pod restarts, and the app reaches it through the internal `postgresql-service`.

## Releasing a New Build
1. Make code changes and commit/push to the repo.
2. CodeBuild automatically builds and pushes a new image tag to ECR (e.g. `1.0.7`).
3. Update the `image:` field in `deployments/coworking.yaml` to the new tag.
4. Apply the change: `kubectl apply -f deployments/coworking.yaml`.
5. Kubernetes performs a rolling update; verify with `kubectl get pods` and check CloudWatch logs.

## Verification
Get the external URL with `kubectl get svc coworking`, then call:
`curl <EXTERNAL-IP>:5153/api/reports/daily_usage` and `/api/reports/user_visits`.

---

## Stand-Out Suggestions

**Resource allocation.** The deployment sets CPU/memory requests and limits (app: 100m/128Mi request, 250m/256Mi limit; DB: 250m/256Mi request, 500m/512Mi limit) so the scheduler can place pods predictably and prevent any single container from starving the node.

**Recommended instance type.** A `t3.small` node is a good fit because the analytics API is a lightweight, low-traffic Python service; its burstable CPU handles periodic report queries cheaply, and it can scale to `t3.medium` only if traffic grows.

**Cost savings.** Costs can be reduced by running a single-node group during development, deleting the EKS cluster when not in use, applying an ECR lifecycle policy to purge old image tags, and setting CloudWatch log retention limits to avoid unbounded log storage.