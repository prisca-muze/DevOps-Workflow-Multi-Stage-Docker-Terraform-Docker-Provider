# Multi-Stage Dockerization and Terraform Provisioning of an Nginx Static Web App


## What is a Multi-Stage Dockerfile?
*A Multi-Stage Dockerfile that separates build stage from runtime stage. You build/prepare things in early stages which is the dependencies stage, then copy only what you need into the final stage — keeping the final image clean and small.*


## Setup Project

- Step 1: `mkdir my-webapp && cd my-webapp`
- Step 2: Create index.html

Type in: 
`code .` on terminal to open VS-Code, then input:

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="index.css">
    <title>My Web App</title>
</head>
<body>
    <div class="card">
        <h1>Welcome to my DevOps Web App!</h1>
        <p>Served by Nginx via Docker and Terraform.</p>
    </div>
</body>
</html>
```


- Step 4. Create a dockerfile file in VS-Code
```
# Use a minimal alpine image just to hold and prep our static file

FROM alpine:latest AS build-stage

# Set the working directory inside this stage's container

WORKDIR /app

# Copy index.html and index.css from your local machine into /app inside this stage

COPY index.html index.css ./  

# Start fresh with a stable, minimal nginx image for the final image

FROM nginx:stable-alpine

# Copy ONLY index.html and index.css from build-stage into nginx's serving folder
# Nothing else from build-stage comes with it

COPY --from=build-stage /app/index.html /app/index.css /usr/share/nginx/html/

# Document that this container listens on port 80

EXPOSE 80

# Start nginx in the foreground (required for Docker — containers need a running process)

CMD ["nginx", "-g", "daemon off;"]
```


- Step 5: Build & push image to dockerhub

```
docker build -t your-username/my-webapp:v2 .
docker login
docker push your-username/my-webapp:v2
```
**Be sure to replace your-username with your actual Docker Hub username.**


## What is Terraform?
*It lets you define infrastructure (containers, servers, networks, etc.) as code in .tf files instead of running manual commands. It's repeatable and version-controlled.*

Terraform commands:
```
- terraform init — downloads the required provider plugins (like the Docker plugin)
- terraform validate - helps you validate terraform configuration files.
- terraform plan — shows you what it will do before doing anything
- terraform apply — actually creates/runs the infrastructure
```


- Step 6: Create terraform folder in the VS Code and a main.tf file inside the folder

Declare Terraform config and specify the Docker provider to download

```bash
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"  # where to download the provider from
      version = "~> 3.9.0"            
    }
  }
}

# Connect to your local Docker daemon (no extra config needed for local)
provider "docker" {}

# Pull the image from Docker Hub
resource "docker_image" "my_webapp_image" {
  name         = "priskah26/my-webapp:v2"  # the image you pushed
  keep_locally = false  # delete the image locally when you run terraform destroy
}

# Create and run a container from that image
resource "docker_container" "my_webapp_container" {
  image = docker_image.my_webapp_image.image_id  # reference the pulled image above
  name  = "my-static-webapp"                 # name of the running container

  ports {
    internal = 80    # port nginx listens on inside the container
    external = 8080  # port you access on your machine: localhost:8080
  }
}
```


- Step 7: Install Terraform (if not installed) in terminal

`sudo apt-get update && sudo apt-get install -y terraform `


- Step 8: Run Terraform in terminal

```
bash
cd terraform
terraform init
terraform plan
terraform apply
```


- Step 9: View your app in your browser

Go to http://localhost:8080 