---
title: Gitness Workshop
authors: Dewan Ahmed & Jim Sheldon
version: 1.2
---

# Gitness Workshop

## Prerequisites

- [Docker installation](https://docs.docker.com/engine/install/) (tested with Docker Desktop 4.26.1 and docker CLI 24.0.7)
- [k3d](https://k3d.io/) installed (tested with 5.6.3)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) installed (tested with 1.28.2)

<details>
  <summary>Optional</summary>
  - [A free account on Harness](https://app.harness.io/auth/#/signup/&?utm_source=website&utm_campaign=devrel&utm_content=gitness-devdays)
  - An image registry with push access, e.g. Docker Hub.
  - An external Kubernetes cluster like GKE, EKS, or DigitalOcean Kubernetes
  - A slack workspace with admin access to create a webhook for notifications.
</details>

## Get Started

### Create a Docker Network

Create a common network for Gitness and the local Kubernetes cluster. This allows Gitness pipeline containers to communicate with the cluster.

```shell
docker network create gitness
docker network ls
```

### Create a local Docker registry

Create a k3d Docker registry to contain the built Docker image for deployment.

```shell
k3d registry create registry.localhost --port 5000
```

### Start Gitness

1. Use the following Docker command to run Gitness (observe the network this container uses):

```shell
docker run -d \
  -e GITNESS_PRINCIPAL_ADMIN_EMAIL=admin@example.com \
  -e GITNESS_PRINCIPAL_ADMIN_PASSWORD=adminpass1 \
  -e GITNESS_USER_SIGNUP_ENABLED=true \
  -e GITNESS_CI_CONTAINER_NETWORKS=gitness \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/gitness:/data \
  --network=gitness \
  --name gitness \
  --restart always \
  harness/gitness
```
2. Once the container is running, open `localhost:3000` in your browser.
3. Select **Sign Up**.
4. Enter a User ID (`developer`), Email (`developer@example.com`), and Password (`devpass1`).
5. Select **Sign Up**. (You might see a warning to change your password. You can ignore that warning.)
6. Log out from the developer account. Click the profile icon from the bottom-left corner and then click **Log out**. 
7. Log in using the admin User ID (`admin`) and Password (`adminpass1`). 

### Create a new Project

