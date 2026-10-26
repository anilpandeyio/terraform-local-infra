
# Run and Destroy Terraform Project

## Run the Project

### 1. Start Colima

```bash
colima start
```

### 2. Configure Docker

```bash
docker context use colima
export DOCKER_HOST="$(docker context inspect colima --format '{{.Endpoints.docker.Host}}')"
```

### 3. Navigate to the project

```bash
cd ~/terraform-local-infra
```

### 4. Initialize Terraform

Run this the first time or when provider configuration changes:

```bash
terraform init
```

### 5. Validate and deploy

```bash
terraform validate
terraform plan
terraform apply
```

Type `yes` when prompted.

### 6. Verify Nginx

```bash
docker ps
curl http://localhost:8080
```

Open http://localhost:8080 in your browser.

## Destroy the Project

### 1. Preview what will be destroyed

```bash
terraform plan -destroy
```

### 2. Destroy Terraform-managed resources

```bash
terraform destroy
```

Type `yes` when prompted.

### 3. Stop Colima (optional)

```bash
colima stop
```

**Note:** In a new terminal, repeat Step 2 under *Run the Project* if Docker connectivity fails. `terraform destroy` removes resources managed by this configuration, while `colima stop` stops the local Docker environment.
