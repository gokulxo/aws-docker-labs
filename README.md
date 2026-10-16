# aws-docker-labs[README (1).md](https://github.com/user-attachments/files/33555331/README.1.md)
# AWS NACL vs Security Group + Docker Hands-on Labs

Hands-on labs I completed while learning cloud and container fundamentals on AWS.

**Tech used:** AWS VPC, EC2, Network ACL, Security Group, Docker, Linux

---

## Lab 1: NACL vs Security Group

### Goal
Understand how a subnet-level **Network ACL** and an instance-level **Security Group** work together to control traffic.

### Traffic flow
```
Client -> NACL (subnet gate) -> Security Group (instance door) -> EC2
```
Traffic must pass **both** layers to reach the instance.

### Step 1: Launch an EC2 instance
- Launched a `t3.micro` instance in a public subnet
- Attached a Security Group with these inbound rules:

| Type | Port | Source |
|------|------|--------|
| SSH | 22 | My IP (`/32`) |
| HTTP | 80 | My IP (`/32`) |
| Custom TCP | 8000 | My IP (`/32`) |

### Step 2: Create a custom NACL (whitelist approach)
Inbound rules:

| Rule # | Type | Port | Source | Action |
|--------|------|------|--------|--------|
| 100 | SSH | 22 | My IP `/32` | ALLOW |
| 110 | HTTP | 80 | My IP `/32` | ALLOW |
| 120 | Custom TCP | 8000 | My IP `/32` | ALLOW |
| 200 | All traffic | All | 0.0.0.0/0 | DENY |

Outbound rules:

| Rule # | Type | Port | Destination | Action |
|--------|------|------|-------------|--------|
| 100 | Custom TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |

The outbound ephemeral port range is required because **NACLs are stateless**: response traffic needs its own outbound rule.

### Step 3: Test
```bash
# On the EC2 instance
python3 -m http.server 8000
```
Open `http://<EC2-public-IP>:8000` in a browser.

| NACL rule 120 (port 8000) | Security Group | Result |
|---|---|---|
| DENY | ALLOW | Page does not load |
| ALLOW | ALLOW | Page loads |

### Step 4: Observations
- A NACL **DENY** blocks traffic even when the Security Group allows it
- NACL rules are evaluated **by rule number, lowest first**
- With a whitelist NACL, anything not explicitly allowed is denied
- Same Wi-Fi network devices share one public IP, so they all match the same `/32` rule
- Console-based "EC2 Instance Connect" comes from AWS IP ranges, so it is blocked by an SSH rule limited to my IP. PuTTY from my laptop works.

### Learnings
- Defense in depth: NACL (subnet) + Security Group (instance)
- Least privilege: open only the ports needed, only to the sources needed
- Stateless (NACL) vs stateful (Security Group)

---

## Lab 2: Docker on EC2

### Goal
Understand images, containers, port mapping, bind mounts and volumes.

### Step 1: Install Docker
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
docker --version
```

### Step 2: Run a container and map a port
```bash
docker run -itd -p 8080:80 httpd
docker ps
```
`-p 8080:80` maps **host port 8080** to **container port 80**.
Browser: `http://<EC2-public-IP>:8080` shows the default "It works!" page.

### Step 3: Bind mount
```bash
mkdir -p /opt/docker
echo "<h1>Hello Gokul</h1>" > /opt/docker/index.html

docker run -itd -p 8888:80 \
  -v /opt/docker:/usr/local/apache2/htdocs \
  httpd
```
Browser: `http://<EC2-public-IP>:8888` shows the file from the EC2 host.

Editing `/opt/docker/index.html` on the host changes the page immediately, with no container restart.

### Step 4: Volume
```bash
docker run -itd -p 9000:80 \
  -v myvol:/usr/local/apache2/htdocs \
  httpd

docker volume ls
```

### Step 5: Observations

| Concept | What I observed |
|---|---|
| Image vs container | Deleting a container does **not** delete its image |
| Host ports | Two containers cannot share the same host port (`port is already allocated`) |
| Container ports | Both containers can use port 80 inside, because each has its own network space |
| Bind mount | Host folder overrides the container folder, so the original "It works!" page is hidden |
| Volume | An empty volume is filled with the container's original content first, so "It works!" still shows |
| Data after delete | Bind mount and volume data survive container deletion |

### Common errors I hit and fixed
| Error | Cause | Fix |
|---|---|---|
| `docker: command not found` | I was inside a container shell, not the EC2 host | `exit` back to the host |
| `Unable to find image locally` | Image not loaded on this machine | `docker load -i my-img.tar` |
| `port is already allocated` | Another container already used host port 8080 | Remove it or use another host port |
| Mount path wrong | Mounted a folder onto a file path | Mount folder to `/usr/local/apache2/htdocs` |

### Cleanup
```bash
docker rm -f $(docker ps -aq)
docker volume rm myvol
```
Stop the EC2 instance when finished to save free-tier hours.

---

## Concepts studied (notes)

- **CloudFormation:** templates create stacks. **Drift** means the real resources no longer match the template (for example someone suspends S3 versioning in the console). *Detect drift* shows expected vs actual values.
- **S3 versioning:** while enabled, every upload creates a new version. After suspending, earlier versions stay, but new uploads overwrite the current version.
- **CI/CD on AWS:** CodeCommit (repo) -> CodePipeline (flow) -> CodeBuild (build + unit test) -> deploy to EC2 or EKS.

---

## Security notes
- Source IP restricted to my own `/32` for SSH and app ports
- No keys, account IDs or public IPs committed to this repo
- `.gitignore` includes `*.pem` and `*.ppk`

## Screenshots
Add images to a `screenshots/` folder and reference them here:

```markdown
![NACL inbound rules](screenshots/nacl-inbound.png)
![Security Group rules](screenshots/sg-inbound.png)
![docker ps](screenshots/docker-ps.png)
```

## Next steps
- Write my own Dockerfile and push an image to Docker Hub
- Build a CI/CD pipeline with CodePipeline
- Kubernetes basics
