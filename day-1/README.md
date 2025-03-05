# ZeroSploit Training Assignment - Docker Deployment

This assignment demonstrates Dockerizing a simple backend application, creating a custom bridge network, and running a container on it.

## Prerequisites
- Docker installed on your system
- Basic understanding of terminal commands

---

## Step 1: Clone the Repository
```bash
git clone https://github.com/ibrahim-reda-2001/ZeroSpolit-Training.git
cd ZeroSploit-Training
```
## Step 2 : Build DockerFile
```bash
docker build -t <name-image> -f Dockerfile . # . represent build context
```
## Step 3: Create a Custom Bridge Network
```bash
docker network create --driver bridge --subnet 192.168.1.0/24 --gateway 192.168.1.1 <name of network>
```
## Step 4: Run the Container on the Custom Network
```bash
docker run -d \
  --name my-backend-container \
  --network <name of network> \
  -p 3000:3000 \
  backend-app
```
## Step 5: Check container status
```bash
docker ps 
```
***see information of network***
```bash
docker network inspect <name of network>
```