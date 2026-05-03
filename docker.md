
```
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

```
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl status docker
sudo systemctl status containerd
```


How to install docker [https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository] <br />


Add user to docker group <br />
```
sudo usermod -aG docker $USER
```

Reload the user 

```
newgrp docker
```

To list running docker 
```
docker ps -a
```


To run docker compose 
```
docker compose up -d 

docker compose ps -a

 docker compose exec -it servicename bash
```

Copy from host to docker compose service 
```
docker compose cp ./run.sh servicename:/app/
```