# Kubernetes & AWS Deployment Guide

This guide explains how to run the BBDMS PHP application on Kubernetes with:

- Docker images stored in **AWS Elastic Container Registry (ECR)**
- User-uploaded media synchronised to **Amazon S3**
- Configurable Kubernetes manifests stored under `k8s/`

Follow every step in order. The commands assume a Bash-compatible shell; adapt them as needed for PowerShell.

---

## 1. Prerequisites

- AWS account with permissions for **ECR**, **S3**, **IAM**, and your Kubernetes control plane (EKS or self-managed).
- `aws` CLI v2, `kubectl`, `docker`, and (optional) `eksctl`/`helm`.
- A running Kubernetes cluster (≥ v1.24) with storage class for persistent volumes and an ingress controller (e.g., AWS Load Balancer Controller or NGINX Ingress).
- TLS automation (optional) via cert-manager if you want automatic HTTPS certificates.

Set helpful environment variables (replace the placeholders):

```bash
AWS_REGION=us-east-1
AWS_ACCOUNT_ID=123456789012
ECR_REPO=bbdms-web
IMAGE_TAG=v1
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
FULL_IMAGE="${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
S3_BUCKET=change-me-bucket
S3_PREFIX=uploads
K8S_NAMESPACE=bbdms
```

---

## 2. Build & Push the Application Image to AWS ECR

1. **Create (or confirm) the repository:**
   ```bash
   aws ecr create-repository --repository-name "${ECR_REPO}" --region "${AWS_REGION}"
   ```

2. **Authenticate Docker to ECR:**
   ```bash
   aws ecr get-login-password --region "${AWS_REGION}" \
     | docker login --username AWS --password-stdin "${ECR_REGISTRY}"
   ```

3. **Build, tag, and push the image:**
   ```bash
   cd BBDMS-Project-PHP-V2.4
   docker build -t "${FULL_IMAGE}" .
   docker push "${FULL_IMAGE}"
   ```

4. **Update `k8s/web-deployment.yaml` and/or `.env` with the pushed image reference (`${FULL_IMAGE}`).**

---

## 3. Provision S3 for Uploaded Images

1. **Create the bucket and (optionally) enable versioning:**
   ```bash
   aws s3 mb "s3://${S3_BUCKET}" --region "${AWS_REGION}"
   aws s3api put-bucket-versioning \
     --bucket "${S3_BUCKET}" \
     --versioning-configuration Status=Enabled
   ```

2. **Create an IAM policy that allows read/write to the bucket** (save as `s3-policy.json`):

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "s3:PutObject",
           "s3:GetObject",
           "s3:DeleteObject",
           "s3:ListBucket"
         ],
         "Resource": [
           "arn:aws:s3:::change-me-bucket",
           "arn:aws:s3:::change-me-bucket/*"
         ]
       }
     ]
   }
   ```

3. **Attach the policy** either to:
   - An IAM user whose access keys will be stored in the Kubernetes secret `bbdms-aws-secret`, or
   - An IAM role used via IRSA (recommended for production). If you use IRSA, mount the role on the `bbdms-web` service account and remove the static credentials from `secret-aws-sample.yaml`.

---

## 4. Prepare Kubernetes Configuration

### 4.1 Namespace

```bash
kubectl apply -f k8s/namespace.yaml
```

### 4.2 Application config

Edit `k8s/configmap-app.yaml` to reflect your bucket name, region, domain, etc., then apply:

```bash
kubectl apply -f k8s/configmap-app.yaml
```

### 4.3 Database secret

Either edit `k8s/secret-db-sample.yaml` and apply it, or create it from literals:

```bash
kubectl create secret generic bbdms-db-secret \
  --namespace "${K8S_NAMESPACE}" \
  --from-literal=DB_USER=bbdms_user \
  --from-literal=DB_PASS=bbdms_password \
  --from-literal=MYSQL_ROOT_PASSWORD=rootpassword
```

### 4.4 AWS credential secret (if not using IRSA)

```bash
kubectl create secret generic bbdms-aws-secret \
  --namespace "${K8S_NAMESPACE}" \
  --from-literal=AWS_ACCESS_KEY_ID=AKIA... \
  --from-literal=AWS_SECRET_ACCESS_KEY=xxxx \
  --from-literal=AWS_REGION="${AWS_REGION}"
```

### 4.5 Image pull secret for ECR

```bash
kubectl create secret docker-registry ecr-credentials \
  --namespace "${K8S_NAMESPACE}" \
  --docker-server="${ECR_REGISTRY}" \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region ${AWS_REGION})"
```

### 4.6 Database schema ConfigMap

The MySQL StatefulSet expects a ConfigMap named `bbdms-schema` that mounts the initial SQL. Create it directly from the checked-in SQL file:

```bash
kubectl create configmap bbdms-schema \
  --namespace "${K8S_NAMESPACE}" \
  --from-file=bbdms.sql="SQL File/bbdms.sql"
```

> You may regenerate the manifest later with `kubectl get cm bbdms-schema -n ${K8S_NAMESPACE} -o yaml > k8s/mysql-schema-configmap.yaml` if you prefer to store it under version control.

---

## 5. Deploy the Stack

Apply every manifest via kustomize (recommended) or individually:

```bash
kubectl apply -k k8s/
```

This creates:

| File | Purpose |
| ---- | ------- |
| `namespace.yaml` | Dedicated namespace segregation |
| `configmap-app.yaml` | Application-level environment variables |
| `secret-*.yaml` | Samples for DB & AWS credentials |
| `mysql-statefulset.yaml` + `mysql-service.yaml` | HA-ready MySQL with persistent storage |
| `web-deployment.yaml` + `web-service.yaml` | PHP/Apache pods with S3 sync sidecar |
| `phpmyadmin-*.yaml` | Optional phpMyAdmin admin UI |
| `ingress.yaml` | Routes `bbdms.example.com` and `phpmyadmin.bbdms.example.com` |

Watch rollout status:

```bash
kubectl get pods -n "${K8S_NAMESPACE}"
kubectl rollout status deployment/bbdms-web -n "${K8S_NAMESPACE}"
kubectl rollout status statefulset/bbdms-mysql -n "${K8S_NAMESPACE}"
```

---

## 6. Verify Database Seed & Connectivity

1. Confirm the MySQL pod ran the schema init (logs should show `Initializing database`).
2. If you need to re-run the SQL import:
   ```bash
   kubectl delete pod/bbdms-mysql-0 -n "${K8S_NAMESPACE}"
   # the StatefulSet restarts and replays scripts only when /var/lib/mysql is empty.
   ```
   or execute the import manually:
   ```bash
   kubectl exec -it statefulset/bbdms-mysql -n "${K8S_NAMESPACE}" -- \
     /bin/sh -c "mysql -uroot -p$MYSQL_ROOT_PASSWORD $MYSQL_DATABASE < /docker-entrypoint-initdb.d/bbdms.sql"
   ```

3. Port-forward phpMyAdmin if ingress is not ready:
   ```bash
   kubectl port-forward svc/bbdms-phpmyadmin -n "${K8S_NAMESPACE}" 8081:80
   ```

---

## 7. Access Through Ingress / Load Balancer

1. Update `k8s/ingress.yaml` with your real domains and TLS secret.
2. Apply the file and wait for the ingress controller to provision an AWS ALB (or whichever controller you use):
   ```bash
   kubectl apply -f k8s/ingress.yaml
   kubectl get ingress -n "${K8S_NAMESPACE}"
   ```
3. Point your DNS records at the provisioned load balancer. Once DNS propagates, browse to:
   - `https://bbdms.example.com` – application
   - `https://phpmyadmin.bbdms.example.com` – admin UI

---

## 8. How S3 Synchronisation Works

- `initContainer (s3-bootstrap)` downloads existing uploads (under `AWS_S3_PREFIX`) into the shared `emptyDir`.
- The main PHP container reads/writes uploaded images to `/var/www/html/images/uploads`.
- The `s3-sync` sidecar pushes local changes back to S3 every `AWS_SYNC_INTERVAL_SECONDS` seconds and keeps S3 as the source of truth.
- On pod restart, S3 is treated as canonical storage, so no PV is required for user media.

To validate, upload a file via the application, then run:

```bash
aws s3 ls "s3://${S3_BUCKET}/${S3_PREFIX}/" --recursive
```

You should see the uploaded asset.

---

## 9. Optional / Advanced Enhancements

- **Use IRSA** to avoid long-lived AWS keys in secrets:
  1. Create an IAM role with the S3 policy.
  2. Annotate a Kubernetes service account used by `bbdms-web`.
  3. Remove `bbdms-aws-secret` from the deployment and let AWS SDK assume the role automatically.

- **Externalise MySQL** using Amazon RDS or Aurora. Update `configmap-app.yaml` (`DB_HOST`) and drop the in-cluster StatefulSet.

- **Backup** by enabling automated snapshots on the PV (EBS) and enabling S3 versioning + lifecycle rules.

- **Scaling**:
  - Horizontal Pod Autoscaler for `bbdms-web`.
  - Use Amazon ElastiCache for sessions if you scale beyond one replica (the current PHP app stores sessions on disk).

---

## 10. Troubleshooting Checklist

- `ImagePullBackOff`: confirm the ECR image exists and the `ecr-credentials` secret has current auth.
- `CrashLoopBackOff` (MySQL): inspect logs for permission errors or missing ConfigMap `bbdms-schema`.
- `S3 sync failures`: ensure `AWS_S3_BUCKET`, `AWS_REGION`, and AWS credentials/IRSA permissions are correct; check pod logs for the `s3-sync` container.
- `Ingress 404`: verify your ingress controller is running and that DNS points to the provisioned load balancer.

---

With these artifacts and steps, anyone can reproduce a complete Kubernetes deployment, leveraging AWS-managed services for container registry and durable object storage.

