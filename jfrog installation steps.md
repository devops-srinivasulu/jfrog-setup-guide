Hi bro! 🔥 Absolutely. Let's set up **JFrog Artifactory from scratch on your AWS Ubuntu server**, then connect your application server and push and pull your frontend and backend Docker images.

I'll give you the steps in execution order, with commands you can copy and paste. Since you haven't started anything yet, we'll begin with the AWS server and finish by verifying that your images can be pushed to JFrog and pulled onto your application server.

We'll use the official JFrog Artifactory installation approach with Docker Compose, following the [official JFrog installation documentation](https://docs.jfrog.com/installation/docs/docker). ([JFrog Docs][1])

# What we are going to build

Your architecture will look like this:

```text
                   AWS CLOUD
                        |
          +-------------+-------------+
          |                           |
          v                           v
+----------------------+    +----------------------+
| JFrog Server         |    | Application Server    |
| Ubuntu EC2           |    | Ubuntu EC2           |
|                      |    |                      |
| JFrog Artifactory    |    | Frontend application |
| Docker Registry      |    | Backend application   |
|                      |    |                      |
| Stores Docker images |    | Builds Docker images  |
+----------------------+    +----------------------+
          ^                           |
          |                           |
          |     PUSH Docker images    |
          +---------------------------+
          |
          |     PULL Docker images
          +---------------------------+
                        |
                        v
             Docker deployment
```

**Our goal:** Your application server will build the frontend and backend images, push them to JFrog, and then pull those images from JFrog whenever you need to deploy them.

We will create two Docker repositories:

| Repository      | Purpose                                              |
| --------------- | ---------------------------------------------------- |
| `docker-local`  | Stores your frontend and backend Docker images       |
| `docker-remote` | Optional later: caches images pulled from Docker Hub |

For our initial setup, we only need `docker-local`.

---

# PHASE 1 — Prepare your AWS infrastructure

## Step 1: Identify your two EC2 servers

Open the AWS EC2 console.

You should have these two servers:

**Server 1: JFrog server**

* OS: Ubuntu Server 22.04 or 24.04 LTS.
* Purpose: Run JFrog Artifactory.
* Public IP or DNS: Required for initial browser access unless you use an alternative private-access method.
* Storage: Start with at least 100 GiB of EBS storage for this setup, and increase it based on your image sizes and retention requirements.
* Instance size: A practical starting point is 4 vCPUs and 16 GiB RAM, subject to the current JFrog system requirements and your workload.

**Server 2: Application server**

* OS: Ubuntu.
* Purpose: Build, push, pull, and run your application Docker images.
* Existing frontend and backend application code.
* Docker Engine installed.

> **Important:** Artifactory consumes significant memory and disk space. A small EC2 instance may be insufficient, especially when the initial setup and indexing are running.

Check the current [JFrog system requirements](https://docs.jfrog.com/installation/docs/installing-artifactory) before choosing your final instance size. ([JFrog Docs][2])

## Step 2: Configure the JFrog server security group

Go to:

**AWS Console → EC2 → Instances → JFrog server → Security → Security groups**

Add the following inbound rules.

| Type       | Port | Source                            | Purpose                            |
| ---------- | ---: | --------------------------------- | ---------------------------------- |
| SSH        |   22 | Your public IP `/32`              | SSH administration                 |
| Custom TCP | 8082 | Your public IP `/32`              | Initial Artifactory UI access      |
| Custom TCP | 8082 | Application server security group | Application server registry access |

For the third rule, select the application server's security group as the source rather than allowing access from the entire internet.

**Do not open port 8082 to `0.0.0.0/0`.**

If your application server is in the same VPC, private connectivity is preferable. Use its private IP for registry access when your network and Artifactory configuration support it.

If you already have a domain and HTTPS configured, we can later put Artifactory behind an HTTPS reverse proxy.

### Important network distinction

If you access Artifactory over HTTPS, use a valid certificate and the appropriate HTTPS port.

For the initial setup below, we will use HTTP on port 8082. That is suitable for an isolated, temporary lab setup, but **do not send passwords or other credentials over an untrusted network using plain HTTP**.

---

# PHASE 2 — Connect to the JFrog Ubuntu server

## Step 3: SSH into the JFrog server

From your local terminal, connect using your JFrog EC2 public IP.

Replace `YOUR_JFROG_PUBLIC_IP` with the actual address.

```bash
ssh -i your-key.pem ubuntu@YOUR_JFROG_PUBLIC_IP
```

If you are already logged in to the Ubuntu server, continue to the next step.

## Step 4: Update Ubuntu

Copy and paste:

```bash
sudo apt update
sudo apt upgrade -y
```

Install the required utilities:

```bash
sudo apt install -y ca-certificates curl gnupg unzip tar
```

Check the Ubuntu version:

```bash
lsb_release -a
```

Check available memory and disk space:

```bash
free -h
df -h /
```

**Do not continue if the server does not have sufficient resources.** Artifactory needs adequate memory and persistent storage to start and operate reliably.

---

# PHASE 3 — Install Docker and Docker Compose

We will use Docker to run Artifactory.

## Step 5: Install Docker Engine

Run these commands on the **JFrog server**.

First, remove conflicting packages if present:

```bash
sudo apt remove -y docker.io docker-doc docker-compose podman-docker containerd runc || true
```

Add Docker's official repository:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Update the package list:

```bash
sudo apt update
```

Install Docker and the Compose plugin:

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Start Docker:

```bash
sudo systemctl enable --now docker
```

Verify the installation:

```bash
sudo docker --version
```

```bash
sudo docker compose version
```

```bash
sudo docker info
```

If these commands work, Docker is installed successfully.

Official reference: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/).

---

# PHASE 4 — Download and install JFrog Artifactory

## Step 6: Create the installation directory

Run:

```bash
sudo mkdir -p /opt/jfrog
```

```bash
cd /opt/jfrog
```

Download the Artifactory Open Source Docker Compose package.

For this walkthrough, we will use version `7.161.15`, which appears in the official installation documentation. Before production deployment, verify the current supported release and package availability. ([JFrog Docs][1])

Run:

```bash
cd /opt/jfrog
```

```bash
sudo curl -fL -o jfrog-artifactory-oss-7.161.15-compose.tar.gz \
  https://releases.jfrog.io/artifactory/bintray-artifactory/org/artifactory/oss/docker/jfrog-artifactory-oss/7.161.15/jfrog-artifactory-oss-7.161.15-compose.tar.gz
```

Check the downloaded file:

```bash
ls -lh /opt/jfrog/jfrog-artifactory-oss-7.161.15-compose.tar.gz
```

Verify that it is a valid gzip archive:

```bash
gzip -t /opt/jfrog/jfrog-artifactory-oss-7.161.15-compose.tar.gz
```

**If the download returns an HTTP error, stop here.** Do not proceed with a missing or invalid archive; check the current release package link in the official JFrog documentation.

## Step 7: Extract the package

Run:

```bash
cd /opt/jfrog
```

```bash
sudo tar -xzf jfrog-artifactory-oss-7.161.15-compose.tar.gz
```

List the extracted directories:

```bash
ls -lah /opt/jfrog
```

Find the extracted Compose directory:

```bash
find /opt/jfrog -maxdepth 2 -name docker-compose.yaml -o -name docker-compose.yml
```

The package normally contains a directory with the release name.

Enter that directory. For example:

```bash
cd /opt/jfrog/artifactory-oss-7.161.15
```

**Important:** If that exact directory does not exist, use the directory name displayed by the previous `ls` command. Do not guess the directory name.

Check its contents:

```bash
ls -lah
```

You should find files such as `config.sh`, a Compose configuration, and an `.env` file.

## Step 8: Run the official configuration script

JFrog provides a configuration script to create the required directory structure and set permissions.

Run:

```bash
sudo bash ./config.sh
```

Follow the prompts.

For this initial single-node lab installation:

* **Installation directory:** Accept the default, or choose `/opt/jfrog-data` if you want a separate persistent data location.
* **Machine IP address:** Use the actual JFrog server IP appropriate for your deployment.
* **Adding a node to an existing cluster:** Choose `N`.
* **Install PostgreSQL:** Choose `Y` for the bundled database option if you are using the installer for a non-production lab.

**Important:** Bundled PostgreSQL is for getting started. For a production deployment, use an appropriately configured external database and follow JFrog's supported database guidance. Also verify the storage location before continuing.

The official configuration script handles important directory and permission setup. Follow its prompts rather than manually changing permissions across the entire installation. ([JFrog Docs][1])

---
## Generate the master key
Run these commands exactly on your JFrog server:
```
cd /opt/jfrog/artifactory-oss-7.161.15

# Create the security directory if required
sudo mkdir -p /root/.jfrog/artifactory/var/etc/security

# Generate and save a new master key
sudo sh -c 'openssl rand -hex 32 > /root/.jfrog/artifactory/var/etc/security/master.key'

# Set ownership and secure permissions
sudo chown 1030:1030 /root/.jfrog/artifactory/var/etc/security/master.key
sudo chmod 600 /root/.jfrog/artifactory/var/etc/security/master.key
```
Important: Keep this key safe. Do not share its contents. JFrog uses it to encrypt sensitive data.
Verify the key exists
```
sudo ls -l /root/.jfrog/artifactory/var/etc/security/master.key
```
You should see the file with permissions similar to:
```
-rw------- 1 1030 1030 ... master.key
```

# PHASE 5 — Start JFrog Artifactory

## Step 9: Start the containers

First, confirm your current directory:

```bash
pwd
```

It should be the extracted Artifactory Compose directory.

Check the Compose configuration:

```bash
sudo docker compose config --quiet
```

If the command succeeds, start Artifactory:

```bash
sudo docker compose up -d
```

List the containers:

```bash
sudo docker compose ps
```

You should see the Artifactory-related containers starting.

Check Docker:

```bash
sudo docker ps
```

### Step 10: Monitor the startup

Artifactory can take several minutes to initialize, especially on its first start.

Check the logs:

```bash
sudo docker compose logs --tail=100
```

To follow the logs:

```bash
sudo docker compose logs -f
```

Press `Ctrl+C` to exit the log display. This does not stop the containers.

If you see errors, do not continue blindly. Resolve the errors before attempting to configure the registry.

Check the Artifactory health endpoint from the JFrog server:

```bash
curl -i http://127.0.0.1:8082/artifactory/api/system/ping
```

Wait for the service to finish initializing.

The expected healthy response is:

```text
HTTP/1.1 200 OK
```

The response body may contain:

```text
OK
```

If it is not ready yet, wait and run the command again.

---

# PHASE 6 — Open the JFrog web interface

## Step 11: Access Artifactory

Open your browser on your local computer.

Navigate to:

```text
http://YOUR_JFROG_PUBLIC_IP:8082
```

Replace `YOUR_JFROG_PUBLIC_IP` with the JFrog EC2 public IP.

For example, if your server's public IP were `203.0.113.10`, you would open:

```text
http://203.0.113.10:8082
```

The IP above is an example, not a real server address.

If the page does not open:

1. Verify the EC2 security-group rules.
2. Check `sudo docker compose ps`.
3. Check the Artifactory logs.
4. Verify that port 8082 is listening:

```bash
sudo ss -lntp | grep 8082
```

## Step 12: Complete the initial login

Follow the initial setup screen.

If the installation asks you to configure an administrator password, create a strong password and save it securely.

If the UI displays the default administrator login, follow the prompts to change the initial password.

**Do not assume the default credentials will work for every installation.** The initial credentials and setup process can depend on the installation method and release.

Once logged in, confirm that the Artifactory UI is working.

---

# PHASE 7 — Create a Docker repository in JFrog

Now we will configure JFrog to store your Docker images.

## Step 13: Create `docker-local`

Inside the Artifactory web interface:

1. Open **Administration**.
2. Select **Repositories**.
3. Click **Create a Repository**.
4. Select **Local**.
5. Select the **Docker** package type.
6. Enter the repository key:

```text
docker-local
```

7. Set the Docker API version to **V2**.
8. Save the repository.

JFrog supports local repositories for storing your organization's own packages and Docker images. The repository creation process is documented in the [official Docker Repositories guide](https://docs.jfrog.com/artifactory/docs/docker-repositories). ([JFrog Docs][3])

### Repository configuration

Your configuration should look like this:

| Setting            | Value                             |
| ------------------ | --------------------------------- |
| Repository type    | Local                             |
| Package type       | Docker                            |
| Repository key     | `docker-local`                    |
| Docker API version | V2                                |
| Purpose            | Store frontend and backend images |

Do not enable anonymous access simply to make pushing images easier. Use authenticated access.

## Step 14: Configure Docker access

For the initial setup, we will use Artifactory's **Repository Path** access method.

In the Artifactory UI, look for:

**Administration → Artifactory → HTTP Settings → Docker Settings**

Select:

```text
Repository Path
```

For the self-managed, no-reverse-proxy setup, JFrog documents the repository-path method and the use of embedded Tomcat. ([JFrog Docs][4])

Save the configuration if the UI requires it.

**Important:** Repository-path access over HTTP requires additional Docker client configuration. We will configure that explicitly below.

---

# PHASE 8 — Configure Docker to communicate with JFrog

For this initial lab setup, we will use the following registry address:

```text
YOUR_JFROG_PRIVATE_IP:8082
```

Using the private IP is preferable when both EC2 instances can communicate over the same VPC.

Replace the placeholder with the actual private IP of your JFrog server.

## Step 15: Configure the Docker daemon on the application server

Connect to your **application server**.

Check Docker:

```bash
sudo docker --version
```

If Docker is not installed, install it using the official [Docker Ubuntu installation instructions](https://docs.docker.com/engine/install/ubuntu/).

Next, configure Docker to allow HTTP access to the JFrog registry for this isolated lab.

Replace `YOUR_JFROG_PRIVATE_IP` in the following commands.

Create a backup of the existing Docker daemon configuration:

```bash
sudo mkdir -p /etc/docker
```

```bash
if [ -f /etc/docker/daemon.json ]; then
  sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.backup
fi
```

**Important:** Do not overwrite an existing daemon configuration that contains settings your application relies on. If you already have a `daemon.json`, merge the required registry entry into the existing JSON instead.

If the file does not exist, create it:

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "insecure-registries": [
    "YOUR_JFROG_PRIVATE_IP:8082"
  ]
}
EOF
```

Replace `YOUR_JFROG_PRIVATE_IP` with the actual private IP before running the command.

Validate the JSON:

```bash
python3 -m json.tool /etc/docker/daemon.json
```

Restart Docker:

```bash
sudo systemctl restart docker
```

Verify the setting:

```bash
sudo docker info | grep -A 5 "Insecure Registries"
```

You should see your JFrog registry address listed.

> **Security warning:** The `insecure-registries` setting is for this initial HTTP lab setup. It is not the recommended final configuration for production. Once HTTPS is configured with a valid certificate, remove the insecure-registry entry and use TLS.

---

# PHASE 9 — Create a user for Docker image access

## Step 16: Create a dedicated JFrog user

For a better security setup, create a separate user for Docker image operations rather than using the administrator account for everyday pushes and pulls.

In Artifactory:

1. Open **Administration**.
2. Go to **User Management**.
3. Create a new user.
4. Choose a username, for example:

```text
docker-publisher
```

5. Set a strong password.
6. Grant only the permissions required for your Docker repository.

For the initial test, you may use your administrator account if necessary, but switch to a dedicated user with appropriately scoped permissions before production use.

JFrog supports authentication through `docker login` and private Docker repositories. See the [official Docker repository guide](https://docs.jfrog.com/artifactory/docs/docker-repositories). ([JFrog Docs][3])

---

# PHASE 10 — Test the Docker registry connection

## Step 17: Log in to JFrog from the application server

On the **application server**, run:

```bash
sudo docker login YOUR_JFROG_PRIVATE_IP:8082
```

Replace the placeholder with the JFrog server's private IP.

Enter your JFrog username and password when prompted.

A successful login should show:

```text
Login Succeeded
```

If the login fails, check:

* Whether the JFrog service is healthy.
* Whether port 8082 is accessible from the application server.
* Whether the Docker daemon includes the registry in its insecure-registry list.
* Whether the Docker access method is configured correctly.
* Whether your credentials are valid.

Do not proceed to push until login succeeds.

---

# PHASE 11 — Push your frontend and backend images

Now we come to your actual requirement.

You mentioned that your application is already running on a separate server and contains both frontend and backend components.

We will use these example image names:

```text
frontend
backend
```

You can replace these with your actual image names.

## Step 18: Identify your existing Docker images

On your application server, run:

```bash
sudo docker images
```

Example output:

```text
REPOSITORY        TAG       IMAGE ID
my-frontend       latest    abc123456789
my-backend        latest    def123456789
```

The names above are examples.

Identify the actual frontend and backend image names and tags from your output.

### Step 19: Tag the frontend image

Suppose your frontend image is:

```text
my-frontend:latest
```

Run:

```bash
sudo docker tag my-frontend:latest YOUR_JFROG_PRIVATE_IP:8082/docker-local/frontend:1.0.0
```

### Step 20: Tag the backend image

Suppose your backend image is:

```text
my-backend:latest
```

Run:

```bash
sudo docker tag my-backend:latest YOUR_JFROG_PRIVATE_IP:8082/docker-local/backend:1.0.0
```

**Replace the source image names and the JFrog IP with the actual values from your environment.**

Check the new tags:

```bash
sudo docker images
```

You should see the JFrog-tagged images.

## Step 21: Push the frontend image

Run:

```bash
sudo docker push YOUR_JFROG_PRIVATE_IP:8082/docker-local/frontend:1.0.0
```

Wait for the push to finish.

## Step 22: Push the backend image

Run:

```bash
sudo docker push YOUR_JFROG_PRIVATE_IP:8082/docker-local/backend:1.0.0
```

Wait for the push to finish.

### Expected result

Both image pushes should complete without errors.

Your repository will contain these images:

```text
docker-local/
├── frontend/
│   └── 1.0.0
└── backend/
    └── 1.0.0
```

You can inspect the repository in the Artifactory UI.

JFrog documents the Docker image naming format and push operation in its [Docker Repositories guide](https://docs.jfrog.com/artifactory/docs/docker-repositories). ([JFrog Docs][3])

---

# PHASE 12 — Pull your images from JFrog

Now we will verify that the images can be downloaded from the registry.

## Step 23: Remove the local copies for testing

Only do this on a machine where removing the local image copies is safe.

For example, on the application server:

```bash
sudo docker image rm YOUR_JFROG_PRIVATE_IP:8082/docker-local/frontend:1.0.0
```

```bash
sudo docker image rm YOUR_JFROG_PRIVATE_IP:8082/docker-local/backend:1.0.0
```

If Docker reports that an image is in use by a container, do not stop or remove a running application just for this test. You can perform the pull test on another machine instead.

## Step 24: Pull the frontend image

Run:

```bash
sudo docker pull YOUR_JFROG_PRIVATE_IP:8082/docker-local/frontend:1.0.0
```

## Step 25: Pull the backend image

Run:

```bash
sudo docker pull YOUR_JFROG_PRIVATE_IP:8082/docker-local/backend:1.0.0
```

Check the images:

```bash
sudo docker images
```

You should now see the images downloaded from JFrog.

**If both pulls succeed, your basic push-and-pull workflow is working.**

---

# PHASE 13 — Deploy the pulled images

Once the images are in JFrog, you can use them in your application deployment process.

For example, your Docker Compose configuration can reference the JFrog registry directly.

```yaml
services:
  frontend:
    image: YOUR_JFROG_PRIVATE_IP:8082/docker-local/frontend:1.0.0
    restart: unless-stopped

  backend:
    image: YOUR_JFROG_PRIVATE_IP:8082/docker-local/backend:1.0.0
    restart: unless-stopped
```

This is only a minimal image-reference example. Your real Compose file must preserve the existing application configuration, including environment variables, ports, networks, volumes, database connectivity, and other required settings.

Before deploying, ensure the correct JFrog credentials are available to Docker on the target server.

Do not replace your current production Compose file with this minimal example.

---

# PHASE 14 — Troubleshooting common errors

## Error 1: `connection refused`

Test connectivity from the application server:

```bash
curl -v http://YOUR_JFROG_PRIVATE_IP:8082/artifactory/api/system/ping
```

If the connection is refused, check whether Artifactory is running and listening on port 8082.

On the JFrog server:

```bash
sudo docker ps
```

```bash
sudo ss -lntp | grep 8082
```

## Error 2: `connection timed out`

Check the following:

* JFrog EC2 security-group inbound rules.
* Application server security-group rules.
* Whether both servers can communicate over the selected private IP.
* Whether the Ubuntu firewall blocks the connection.
* Whether the JFrog service is running.

## Error 3: `http: server gave HTTP response to HTTPS client`

This usually means Docker is attempting HTTPS while your lab registry is using HTTP.

Verify that the registry address is listed under the Docker daemon's `insecure-registries` configuration, and restart Docker after correcting it.

Do not use the insecure-registry workaround as a substitute for proper HTTPS in production.

## Error 4: `unauthorized: authentication required`

Check:

* Your JFrog username and password.
* Whether the user has access to the repository.
* Whether you logged in to the correct registry host and port.
* Whether the repository key in the image tag is correct.

## Error 5: Docker push fails

Check the Artifactory logs:

```bash
sudo docker compose logs --tail=200
```

Also check available storage:

```bash
df -h
```

If the disk is full, resolve the storage issue before retrying.

---

# PHASE 15 — What your final setup should look like

After completing all the steps, your environment should have the following components.

| Component               | Expected result                     |
| ----------------------- | ----------------------------------- |
| AWS Ubuntu server       | Running JFrog Artifactory           |
| Docker Engine           | Installed on JFrog server           |
| Docker Compose          | Installed and working               |
| Artifactory UI          | Accessible to authorized users      |
| Docker local repository | `docker-local` created              |
| Application server      | Can authenticate to JFrog           |
| Frontend image          | Successfully pushed and pulled      |
| Backend image           | Successfully pushed and pulled      |
| Docker deployment       | Able to reference images from JFrog |

## My recommendation for your production environment

Once the initial lab setup works, I recommend the following improvements before you use JFrog for your production applications:

1. Configure HTTPS with a valid TLS certificate.
2. Restrict registry access to the application server and authorized administrators.
3. Use a dedicated publishing account with limited permissions.
4. Store credentials securely rather than hardcoding them in scripts or Compose files.
5. Configure EBS capacity and backups for Artifactory's persistent data.
6. Configure image retention policies to control storage usage.
7. Integrate image pushes and pulls into Jenkins once the manual workflow is verified.

---

## Bro, before you execute everything, I need three details from your AWS environment.

The commands above establish the workflow, but the registry address and Docker configuration must match your actual infrastructure. I don't want you to copy a placeholder or accidentally affect your running application.

**Please send me these three details:**

**1. JFrog server**

* Have you already launched the Ubuntu EC2 instance?
* If yes, what are its instance type and Ubuntu version?
* Is it in the same VPC as your application server?

**2. Application server**

* Is Docker already installed?
* What are the actual frontend and backend image names from `sudo docker images`?
* Are your applications running through Docker Compose?

**3. Registry access**

* Do you have a domain name available for JFrog, or do you want to start with the EC2 private IP?

With those details, I can tailor the installation and registry commands to your actual servers, including the correct frontend and backend image names, without guessing your environment's values.

[1]: https://docs.jfrog.com/installation/docs/docker?utm_source=chatgpt.com "Artifactory Docker Installation"
[2]: https://docs.jfrog.com/installation/docs/installing-artifactory?utm_source=chatgpt.com "Artifactory Installation"
[3]: https://docs.jfrog.com/artifactory/docs/docker-repositories?utm_source=chatgpt.com "Docker Repositories"
[4]: https://docs.jfrog.com/artifactory/docs/additional-docker-information?utm_source=chatgpt.com "Additional Docker Information"
