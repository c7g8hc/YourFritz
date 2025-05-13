Here's a **complete solution** to back up a MongoDB service exposed via ClusterIP (`gemlab-service`) once a day using a **Kubernetes CronJob**, push the backup image to **Amazon ECR**, and automate the build/push via **GitHub Actions (GHA)**.

---

## 🔧 Assumptions

* MongoDB is reachable via `gemlab-service:27017`.
* You want to backup to a persistent volume (e.g., AWS EBS) or upload to an S3 bucket (I'll include both options).
* Your MongoDB **does not** require authentication (if it does, you can add creds).
* You already have an **ECR repo created** (e.g., `mongo-backup`) and AWS credentials available in GitHub Secrets.

---

## 1️⃣ `Dockerfile` for MongoDB Backup

```Dockerfile
# Dockerfile
FROM mongo:7

# Install AWS CLI if uploading to S3
RUN apt-get update && apt-get install -y awscli

COPY backup.sh /backup.sh
RUN chmod +x /backup.sh

CMD ["/backup.sh"]
```

---

## 2️⃣ `backup.sh` Script

This script backs up MongoDB and uploads to S3. Comment out S3 parts if not needed.

```bash
#!/bin/bash
set -e

TIMESTAMP=$(date +"%F-%H-%M-%S")
BACKUP_DIR="/backup/mongo-backup-$TIMESTAMP"
S3_BUCKET="s3://your-s3-bucket-name"

echo "Starting MongoDB backup at $TIMESTAMP"

mkdir -p "$BACKUP_DIR"

mongodump --host gemlab-service --port 27017 --out "$BACKUP_DIR"

echo "MongoDB backup complete. Directory: $BACKUP_DIR"

# Optional: upload to S3
if [[ -n "$UPLOAD_TO_S3" ]]; then
  echo "Uploading backup to S3..."
  aws s3 cp "$BACKUP_DIR" "$S3_BUCKET/$TIMESTAMP" --recursive
  echo "Upload complete."
fi

echo "Backup process done."
```

---

## 3️⃣ Kubernetes CronJob YAML

```yaml
# k8s/cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongo-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mongo-backup
            image: <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/mongo-backup:latest
            env:
            - name: UPLOAD_TO_S3
              value: "true"
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: mongo-backup-pvc
```

> You also need to create the PVC (`mongo-backup-pvc`) separately or change to use `emptyDir` if you're uploading to S3 immediately.

---

## 4️⃣ GitHub Actions Workflow: `.github/workflows/deploy-backup.yml`

```yaml
name: Build and Deploy MongoDB Backup Image

on:
  push:
    branches: [main]

env:
  ECR_REPOSITORY: mongo-backup
  AWS_REGION: us-east-1

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<GITHUB_ACTIONS_ROLE>
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build Docker image
      run: |
        docker build -t $ECR_REPOSITORY:latest .
        docker tag $ECR_REPOSITORY:latest ${{ steps.login-ecr.outputs.registry }}/$ECR_REPOSITORY:latest

    - name: Push to ECR
      run: |
        docker push ${{ steps.login-ecr.outputs.registry }}/$ECR_REPOSITORY:latest
```

---

## 5️⃣ Optional PVC for Volume (if not using S3)

```yaml
# k8s/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-backup-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

---

## ✅ To Deploy:

1. **Push your Dockerfile + scripts to GitHub**
2. Add GitHub Secrets:

   * `AWS_REGION`
   * `AWS_ACCOUNT_ID`
   * `GITHUB_ACTIONS_ROLE` (OIDC role for GitHub)
3. Apply the PVC:

   ```bash
   kubectl apply -f k8s/pvc.yaml
   ```
4. Apply the CronJob:

   ```bash
   kubectl apply -f k8s/cronjob.yaml
   ```

---
In Rancher Kubernetes Engine (RKE), containers in your cluster may experience permission issues when executing shell scripts due to the user under which they are running. Let’s break this down and look at causes and fixes.

---

### 🔍 **Why Permission Issues Happen**

Containers in Kubernetes (including RKE clusters) **do not necessarily run as root**. The behavior depends on:

1. **The Docker image used**: Some images (like `ubuntu`, `centos`, etc.) default to running as root, but others (like `nginx`, `node`, `alpine`, etc.) may specify a non-root user via the `USER` directive in the Dockerfile.

2. **Pod Security Policies (PSP), Security Context, or OPA/Gatekeeper**: Kubernetes might enforce that containers **run as non-root** or under specific user IDs (UIDs).

3. **File permissions**: Your shell script inside the container might not be executable by the user the container runs as (`chmod` or `chown` issues).

---

### 🧪 **How to Check the User**

1. **Check Dockerfile or Image User**:

   ```sh
   docker inspect <image-name> | grep -i user
   ```

2. **Check Running Pod User**:
   Use `kubectl` to `exec` into the container and run:

   ```sh
   whoami
   id
   ```

3. **Look at Security Context** in the pod YAML:

   ```yaml
   securityContext:
     runAsUser: 1000
     runAsNonRoot: true
   ```

---

### 🛠️ **How to Fix It**

Depending on the issue, here are solutions:

#### 1. ✅ **Script Permission Fix**

Make sure your script inside the container is executable:

```Dockerfile
COPY your-script.sh /usr/local/bin/your-script.sh
RUN chmod +x /usr/local/bin/your-script.sh
```

#### 2. ✅ **Run as Root (If Allowed)**

If your container needs root for specific actions, you can add this to your pod YAML:

```yaml
securityContext:
  runAsUser: 0
```

> **Note**: This might be restricted by your cluster’s security policies (e.g., PSP, OPA).

#### 3. ✅ **Use Custom Security Context**

If you can’t run as root, ensure the script is owned by and executable by the non-root user:

```Dockerfile
RUN adduser --disabled-password --gecos '' appuser && \
    chown appuser:appuser /usr/local/bin/your-script.sh && \
    chmod +x /usr/local/bin/your-script.sh

USER appuser
```

#### 4. ✅ **Update Pod YAML**

Ensure the correct user is specified in your Kubernetes manifest:

```yaml
containers:
- name: example
  image: your-image
  command: ["/bin/sh", "-c", "/usr/local/bin/your-script.sh"]
  securityContext:
    runAsUser: 1001
```

---

### 📌 Summary

| Reason                      | Symptom                      | Fix                                  |
| --------------------------- | ---------------------------- | ------------------------------------ |
| Non-root container user     | Permission denied errors     | Use `securityContext`, fix ownership |
| Script not executable       | `/bin/sh: permission denied` | Use `chmod +x` in Dockerfile         |
| Cluster policy restrictions | Fails to run as root         | Adjust PSP/OPA, or adapt image       |

