# Project Workflow: Provisioning, Updating, and Destroying

This guide explains what Terraform does when it runs this project and how to
safely manage the local Docker resources over their lifecycle. For the short
command checklist, see [README.md](./README.md).

## What this project manages

The configuration in `main.tf` declares three important pieces of information:

1. Terraform must use the `kreuzwerker/docker` provider, constrained to the
   `~> 3.0` version series. `.terraform.lock.hcl` records the selected provider
   version and checksums so future initializations can use the same version.
2. The Docker provider connects to the Docker daemon available to the current
   shell. On this Mac, that is Colima's Docker daemon.
3. Terraform manages a Docker network named `terraform-app-network` and an
   NGINX container named `terraform-nginx`. The container uses
   `nginx:alpine`, joins the managed network, and maps host port `8080` to
   container port `80`.

The host port is the entry point for the local browser or `curl` request:

```text
Browser/curl -> localhost:8080 -> Docker port mapping -> NGINX:80
```

The configuration runs on the local Docker host; it does not create a cloud
server, configure a public endpoint, or install NGINX directly on macOS.

## How Terraform decides what to do

Terraform compares the desired resources described in `main.tf` with the
resources recorded in its state and, during planning/applying, information
available from the Docker provider. From that comparison it determines whether
to create, update, replace, or destroy resources.

Terraform state is essential to this process: it maps configuration addresses
such as `docker_container.nginx` to the real objects managed in Docker. Keep
the state file alongside the project between runs. Do not edit it by hand or
delete it to try to reset the deployment. State is local by default in this
project and may contain sensitive infrastructure details.

The container refers to the network resource, so Terraform creates the network
before the container. When destroying, Terraform removes the container before
removing the network it depends on.

## 1. Prepare the local Docker connection

Start Colima and configure the current terminal to use its Docker context:

```sh
colima start
docker context use colima
export DOCKER_HOST="$(docker context inspect colima --format '{{.Endpoints.docker.Host}}')"
```

Confirm that the Docker CLI can reach the daemon:

```sh
echo "$DOCKER_HOST"
docker version
```

The `DOCKER_HOST` value should point to the Colima socket. The environment
variable is set for the current terminal only. In a new terminal, configure
the Docker connection again if needed. Terraform commands should be run from
that same configured terminal so the Docker provider connects to the intended
daemon.

## 2. Initialize the working directory

Run this before the first plan/apply in a checkout, or after changing provider
requirements:

```sh
terraform init
```

Initialization downloads the provider and prepares Terraform's local working
directory. It does not create the network or container. Keep
`.terraform.lock.hcl` under version control; do not commit the downloaded
`.terraform/` working directory.

## 3. Validate and review a plan

Before making infrastructure changes, validate the configuration and inspect
the proposed actions:

```sh
terraform validate
terraform plan
```

Validation checks the Terraform configuration for structural and provider
errors. Planning compares the declared resources to Terraform's state and
shows the proposed changes without applying them. Review the plan and confirm
that it targets the intended local Docker daemon and resources.

For a fresh deployment, the plan should propose creating the network and
container. If Terraform proposes unexpected replacements or removals, stop and
investigate before approving an apply.

## 4. Apply the configuration

Create or update the resources with:

```sh
terraform apply
```

Terraform shows a plan and asks for confirmation. Enter `yes` only after
reviewing the actions. Terraform then asks the Docker provider to make the real
Docker resources match the configuration and updates its state to record the
result. The `nginx:alpine` image may need to be downloaded if it is not already
available to the daemon.

Verify the service and managed objects:

```sh
curl http://localhost:8080
docker ps --filter name=terraform-nginx
docker network inspect terraform-app-network
terraform state list
```

The HTTP request should return the NGINX welcome page. `terraform state list`
shows the Terraform resource addresses currently tracked in state.

## 5. Change and redeploy

Make infrastructure changes in `main.tf`, then inspect and apply them:

```sh
terraform fmt
terraform validate
terraform plan
terraform apply
```

For example, changing `ports.external` changes the host-side port used to
reach NGINX; the container continues to listen on port `80`. If port `8080` is
already occupied, choose a different available host port and use that port in
the verification URL.

Terraform may update a resource in place or replace it, depending on the
change and provider behavior. Always review the plan. Changes made manually
outside Terraform can create drift from the declared configuration; use a new
plan to see what Terraform proposes to bring back in line.

## 6. Destroy the deployment

To preview removal without changing anything:

```sh
terraform plan -destroy
```

When you are ready to remove the project resources, run:

```sh
terraform destroy
```

Review the proposed destruction and enter `yes` to confirm. Terraform removes
the container and then the network, and updates its state to record that those
resources are no longer managed. This does not uninstall Terraform, remove
Colima, or delete the project files.

Afterward, the service at `http://localhost:8080` should no longer respond.
You can stop the local Docker environment separately when you no longer need
it:

```sh
colima stop
```

Stopping Colima is not the same as destroying the Terraform resources. It
stops the Docker environment; the resources may still be present when Colima
starts again. Run `terraform destroy` while the daemon is available if you
intend to remove them.

## Important operating notes

- Run Terraform against the same Docker daemon and with the same state that
  were used to create the resources. Switching Docker contexts can make a
  different set of containers visible to the Docker provider.
- Do not remove the Terraform-managed container or network manually with
  `docker rm` or `docker network rm` as a routine cleanup. Use Terraform so
  the real resources and Terraform state stay consistent.
- Do not delete or commit `terraform.tfstate`, `terraform.tfstate.backup`,
  Terraform plan files, or secret variable files. State and plans can expose
  sensitive values. The repository's `.gitignore` excludes these local files.
- If Terraform reports that it cannot connect to Docker, first start Colima
  and repeat the Docker context and `DOCKER_HOST` setup in the current
  terminal, then retry the Terraform command.
