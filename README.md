# Jenkins-Setup-Docker
Jenkins-Setup-Docker


## Create a directory Jenkins
```bash
mkidr Jenkins
mkdir -p Jenkins/Jenkins_DockerFile
```

## Create Dockerfile in Jenkins_DockerFile Folder
```bash
cd Jenkins/Jenkins_DockerFile
vi Dockerfile
```

```bash
FROM jenkins/jenkins:lts

USER root

# Install dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
      wget \
      curl \
      ca-certificates \
      gnupg \
      lsb-release \
      apt-transport-https \
    && rm -rf /var/lib/apt/lists/*

# Add Docker repo (Debian, not Ubuntu)
RUN install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
    && chmod a+r /etc/apt/keyrings/docker.asc \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
       https://download.docker.com/linux/debian $(lsb_release -sc) stable" \
       | tee /etc/apt/sources.list.d/docker.list > /dev/null

# Add Trivy repo
RUN wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
      | gpg --dearmor > /usr/share/keyrings/trivy.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
       https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" \
       | tee /etc/apt/sources.list.d/trivy.list

# Install Docker + Trivy
RUN apt-get update && apt-get install -y --no-install-recommends \
      docker-ce \
      docker-ce-cli \
      containerd.io \
      docker-buildx-plugin \
      docker-compose-plugin \
      trivy \
    && rm -rf /var/lib/apt/lists/*

# Install Gitleaks
RUN GITLEAKS_VERSION=$(curl -s https://api.github.com/repos/gitleaks/gitleaks/releases/latest \
        | grep -Po '"tag_name": "v\K[0-9.]+' ) \
    && curl -sSL "https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz" -o /tmp/gitleaks.tar.gz \
    && tar -xzf /tmp/gitleaks.tar.gz -C /usr/local/bin gitleaks \
    && rm -rf /tmp/gitleaks.tar.gz


# Install kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
    && install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl \
    && rm kubectl

# Add Jenkins to docker group
RUN usermod -aG docker jenkins

USER jenkins
```

## Create Docker Compose File in Jenkins Directory
```bash
cd Jenkins
vi docker-compose.yaml
```


```bash
version: "3.9"

services:
  jenkins:
    build:
      context: ./Jenkins_DockerFile
      dockerfile: Dockerfile
    container_name: jenkins
    restart: unless-stopped
    user: root
    ports:
      - "8081:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - cicd-net

  sonarqube:
    image: sonarqube:latest
    container_name: sonarqube
    restart: unless-stopped
    ports:
      - "9000:9000"
    environment:
      - SONARQUBE_JDBC_USERNAME=sonar
      - SONARQUBE_JDBC_PASSWORD=sonar
      - SONARQUBE_JDBC_URL=jdbc:postgresql://db:5432/sonar
    depends_on:
      - db
    networks:
      - cicd-net

  db:
    image: postgres:13
    container_name: sonarqube-db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
      - POSTGRES_DB=sonar
      - PGDATA=/var/lib/postgresql/data/pgdata
    volumes:
      - postgresql:/var/lib/postgresql/data
    networks:
      - cicd-net

volumes:
  jenkins_home:
  postgresql:

networks:
  cicd-net:
    driver: bridge
```


Apply
```bash
docker compose up -d --build
```
