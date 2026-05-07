
## 📋 Table of Contents

- [Level 1 — Cloud Deployment](#level-1--cloud-deployment)
- [Level 2 — Monitoring & Observability](#level-2--monitoring--observability)
- [Level 3 — CI/CD Pipeline with Jenkins](#level-3--cicd-pipeline-with-jenkins)
- [Port Reference](#port-reference)

---


## Level 1 — Cloud Deployment

### Step 1 — Dockerfiles for Frontend & Backend

- **Backend:** `Dockerfile` using `python:3.11-slim` as base image, running the FastAPI app via `uvicorn`.
- **Frontend:** Separate `Dockerfile` using `nginx` to serve the static HTML/CSS internship page.

Both images were tested to build cleanly before deployment.
<img width="667" height="347" alt="image" src="https://github.com/user-attachments/assets/c0f71f29-de0d-44ad-aadf-760e672a71d0" />
<img width="758" height="420" alt="image" src="https://github.com/user-attachments/assets/33113dce-4b53-4ffc-bbd4-e8c9cd490727" />


---

### Step 2 — docker-compose.yml & Local Testing

All six services were composed into a single `docker-compose.yml`:

- `nginx_frontend`
- `fastapi_backend`
- `prometheus`
- `grafana`
- `node_exporter`
- `blackbox_exporter`
~~~
version: "3.8"

services:

  backend:
    build: ./backend
    image: india0/fusionpact_backend:latest
    container_name: fusionpact_backend
    ports:
      - "8000:8000"
    volumes:
      - /mnt/efs/backend_data:/app/app/data
    restart: unless-stopped
    networks:
      - app_network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/"]
      interval: 30s
      timeout: 10s
      retries: 3

  frontend:
    build: ./frontend
    image: india0/fusionpact_frontend:latest
    container_name: fusionpact_frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    restart: unless-stopped
    networks:
      - app_network

  prometheus:
    image: prom/prometheus:latest
    container_name: fusionpact_prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"
      - "--web.enable-lifecycle"
    restart: unless-stopped
    networks:
      - app_network

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus
    restart: unless-stopped
    networks:
      - app_network

  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    restart: unless-stopped
    networks:
      - app_network

  blackbox_exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox_exporter
    ports:
      - "9115:9115"
    volumes:
      - ./monitoring/blackbox.yml:/etc/blackbox_exporter/config.yml:ro
    restart: unless-stopped
    networks:
      - app_network


volumes:
  prometheus_data:
  grafana_data:


networks:
  app_network:
    driver: bridge
~~~
Ran `docker compose up` locally and verified that the frontend and backend were reachable before pushing to EC2.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/233c0dcf-d508-42db-a5fa-61aa2e7976c3" />


---

### Step 3 — EC2 Instance on AWS

| Setting | Value |
|---|---|
| Instance Type | `t3.small` |
| OS | Ubuntu 22.04 LTS |
| Region | `ap-south-1` |
<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/42c4e9f8-f303-4a16-b1a0-b925b9be531f" />

**Security Group Inbound Rules:**

| Port | Purpose |
|---|---|
| 22 | SSH |
| 80 | Frontend (Nginx) |
| 8000 | Backend (FastAPI) |
| 9090 | Prometheus |
| 3000 | Grafana |
| 9100 | Node Exporter |
| 9115 | Blackbox Exporter |
| 8080 | Jenkins |

---

### Step 4 — EFS for Persistent Storage

- Created an **AWS EFS** file system in the same VPC as the EC2 instance.
- Installed `nfs-common` and mounted the EFS manually using the NFS protocol.
- Mounted EFS at `/mnt/efs`.
- Created `/mnt/efs/backend_data` and mapped it into the backend container through `docker-compose.yml`.
<img width="1920" height="1032" alt="image" src="https://github.com/user-attachments/assets/13b8fc11-8ac7-43a9-bb23-7c5766c51f04" />

## Why EFS Was Chosen Over Alternatives

The backend application stores data inside JSON files, so persistent storage was required to ensure data survives container restarts and EC2 instance replacement.

Several storage options were evaluated before selecting AWS EFS.

### Docker Volumes

Docker volumes provide persistent storage only on the local host machine.  
They work well for single-instance development environments, but the data remains tied to that EC2 instance.

If the EC2 instance is:
- terminated,
- replaced,
- corrupted,
- or migrated,

the Docker volume data may also be lost unless additional backup mechanisms are configured.

Docker volumes also make scaling and migration more difficult because the storage is not shared across multiple instances by default.

---

### Amazon RDS

Amazon RDS is a managed relational database service designed for structured databases such as:
- MySQL
- PostgreSQL
- MariaDB

However, this project stores lightweight JSON-based data

Using RDS for this use case would introduce:
- unnecessary complexity,
- higher operational overhead,
- and additional cost.

For a lightweight file-based backend, RDS would be over-engineered.

---

### AWS EFS (Chosen Solution)

AWS EFS was selected because it provides:
- persistent shared storage,
- automatic scalability,
- managed infrastructure,
- and easy integration with EC2.

EFS remains independent of the EC2 instance lifecycle, meaning data survives:
- container recreation,
- Docker restarts,
- and even EC2 replacement.

It also integrates easily with Linux systems using NFS mounts, making it ideal for containerized workloads that require shared persistent storage.

---

## Storage Comparison Table

| Feature | Docker Volumes | Amazon RDS | AWS EFS ✅ |
|---|---|---|---|
| Persistent Across Container Restart | ✅ Yes | ✅ Yes | ✅ Yes |
| Persistent Across EC2 Replacement | ❌ No | ✅ Yes | ✅ Yes |
| Shared Across Multiple Instances | ❌ No | ✅ Yes | ✅ Yes |
| Easy Docker Integration | ✅ Yes | ⚠️ Moderate | ✅ Yes |
| Operational Complexity | Low | High | Medium |
| Cost for Small File Store | Low | Higher | Cost-Effective |
| Best Use Case | Local container persistence | Relational databases | Shared persistent file storage |

```

### Step 5 — Deploy on EC2

```bash
# SSH into EC2
ssh -i <key.pem> ubuntu@<EC2_PUBLIC_IP>

# Clone repository
git clone <repo-url>
cd <repo>

# Deploy all containers
docker compose up -d --build
```
<img width="1095" height="314" alt="image" src="https://github.com/user-attachments/assets/a058c69d-1518-41ed-9bf6-0a9ed51827a0" />


All **6 containers** came up healthy. Frontend and backend were publicly accessible via the EC2 public IP.
<img width="1920" height="1032" alt="image" src="https://github.com/user-attachments/assets/a744eec0-a089-4aee-a3fb-c14df4f4511e" />
<img width="562" height="245" alt="image" src="https://github.com/user-attachments/assets/cc5b0da4-7374-4fa3-b4c3-7346adc6d874" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/6e634960-6ea1-4801-b423-0ce1babd36f6" />


---

## Level 2 — Monitoring & Observability

### Step 6 — Prometheus Scrape Targets

Configured `prometheus.yml` with four scrape jobs:

| Job | Target | Purpose |
|---|---|---|
| `prometheus` | `localhost:9090` | Self-monitoring |
| `fastapi_backend` | `:8000/metrics` | Application metrics |
| `node_exporter` | `:9100` | EC2 host metrics |
| `blackbox_exporter` | `:9115` | HTTP probe for frontend, backend root, and `/users` endpoint |
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/9301e641-bad5-4922-8e7d-8aa917f6aef7" />

---

### Step 7 — Blackbox Exporter Configuration

Created `blackbox.yml` defining the `http_2xx` probe module.

Blackbox Exporter **actively probes** each endpoint from outside the container and reports:
- Up/Down status
- HTTP response latency


---

### Step 8 — Grafana Setup

1. Opened Grafana at `http://<EC2_IP>:3000`
2. Logged in with admin credentials
3. Added **Prometheus** (`http://prometheus:9090`) as the data source
4. Verified the connection was successful

---

### Step 9 — Node Exporter Dashboard (Infrastructure Metrics)

Imported community dashboard **ID 1860** — *Node Exporter Full*.

Panels include:
- Real-time EC2 **CPU usage**
- **Memory utilisation**
- **Disk I/O**
- **Network traffic**
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a6fe84c6-cb16-43e6-8f3c-150f75647318" />


---

### Step 10 — Blackbox Exporter Dashboard (Endpoint Monitoring)


Panels include:
- Endpoint **up/down status**
- HTTP **response time**
- **Probe success rate** for all monitored URLs
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/e86dfce2-3e9c-4ec2-a4d2-80b330635a85" />

---

### Step 11 — Custom Application Dashboard

Built a custom Grafana dashboard using FastAPI's built-in `/metrics` data.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/fe934481-1673-42dc-8bad-8bbaa83f609c" />



---

## Level 3 — CI/CD Pipeline with Jenkins

### Step 12 — Installed Jenkins on EC2

Installed Jenkins on the Ubuntu 22.04 EC2 instance.

### Jenkins Installation

```bash
sudo apt update

sudo apt install openjdk-17-jdk -y

### Step 13 — Initial Jenkins Setup

```bash
# Retrieve initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

- Logged in at `:8080`
- Installed **suggested plugins**
- Additionally installed the **SSH Agent** plugin

---

### Step 14 — Jenkins Credentials

Stored the following under **Manage Jenkins → Credentials → Global**:

| Credential ID | Type | Purpose |
|---|---|---|
| `dockerhub_username` | dockerhub username |
| `dockerhub_acesskey` | personal access token created in dockerhub account |


---

### Step 15 — Jenkinsfile

Go to jenkins ui select new item -> pipeline and paste this pipeline inside :

```groovy
pipeline {

    agent any

    environment {

        PROJECT_DIR = "/home/ubuntu/fusionpact-devops-challenge"
    }

    stages {

        stage('Update Code') {

            steps {

                dir("${PROJECT_DIR}") {

                    sh '''
                    git pull origin main
                    '''
                }
            }
        }

        stage('Build Docker Images') {

            steps {

                dir("${PROJECT_DIR}") {

                    sh '''
                    docker compose build
                    '''
                }
            }
        }

        stage('Push Docker Images') {

            steps {

                dir("${PROJECT_DIR}") {

                    withDockerRegistry([credentialsId: 'dockerhub-creds', url: '']) {

                        sh '''
                        docker compose push
                        '''
                    }
                }
            }
        }

        stage('Deploy Application') {

            steps {

                dir("${PROJECT_DIR}") {

                    sh '''
                    docker compose up -d --build

                    docker image prune -f
                    '''
                }
            }
        }
    }

    post {

        success {

            echo 'Pipeline executed successfully!'
        }

        failure {

            echo 'Pipeline failed!'
        }
    }
}
```

---

### Step 16 — Jenkins Pipeline Job
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/3f3b4ccd-b4d0-430a-99d2-0302f878006d" />

---

## Port Reference

| Service | Container | Port | Purpose |
|---|---|---|---|
| Frontend | `nginx_frontend` | `80` | Nginx serving HTML/CSS internship page |
| Backend | `fastapi_backend` | `8000` | FastAPI REST API + `/metrics` endpoint |
| Prometheus | `prometheus` | `9090` | Metrics collection & storage |
| Grafana | `grafana` | `3000` | Dashboards & visualisation |
| Node Exporter | `node_exporter` | `9100` | EC2 host CPU / RAM / Disk metrics |
| Blackbox Exporter | `blackbox_exporter` | `9115` | HTTP endpoint up/down probing |
| Jenkins | `jenkins` | `8080` | CI/CD pipeline execution |
| EFS Mount | — | — | `/mnt/efs/backend_data` — persistent JSON store |

---

