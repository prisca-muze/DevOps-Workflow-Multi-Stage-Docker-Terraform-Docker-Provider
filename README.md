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

## ADDITIONS I MADE IN THIS PROJECT
I made my static webpage to be more visually pleasing using an external css

## ERRORS I RAN INTO
- Error 1: My Docker was outdated, had to update.
- Error 2: I typed in : `docker build -t priskah26/my-webapp:v1 `
                               
I received this error: 
```
ERROR: docker: 'docker buildx build' requires 1 argument
Usage:  docker buildx build [OPTIONS] PATH | URL | -
Run 'docker buildx build --help' for more information
```
So, I corrected it to: `docker build -t priskah26/my-webapp:v1 .`   

Note that the dot (.) at the end of the command symbolizing the current folder I want the docker to build in.

- Error 3: I did `terraform init`, then `terraform plan`

And this error showed: 
```
╷
│ Error: No configuration files
│
│ Plan requires configuration to be present. Planning without a configuration would mark everything for destruction, which is
│ normally not what is desired. If you would like to destroy everything, run plan with the -destroy option. Otherwise, create a
│ Terraform configuration file (.tf file) and try again.
╵
```
I forgot to go into the terraform directory to run those commands and I did so using `cd terraform`.

- Error 4: I did `terraform apply`
And this error showed:
```
╷
│ Error: Unable to read Docker image into resource: unable to pull image priskah26/static-my-webapp:v1: error pulling image priskah26/static-my-webapp:v1: Error response from daemon: pull access denied for priskah26/static-my-webapp, repository does not exist or may require 'docker login'
│
│   with docker_image.my_webapp_image,
│   on main.tf line 15, in resource "docker_image" "my_webapp_image":
│   15: resource "docker_image" "my_webapp_image" {
│
╵
```
The resource name in my terraform/main.tf was different from the name I pushed. So, I made sure they tallied.

- Error 5: When I had added my external index.css, it didn’t reflect on my browser because I forgot to add it in my dockerfile build and final stages in my VS Code. I thought I had it done and went through the process of `docker build` - `docker push` - `terraform init` - `terraform plan` - `terraform apply` and the css syling didn’t display. 

- Error 6: So, I went back to docker to inspect and tried to run my docker locally with `docker inspect (container ID)`, `docker run -d --name [container-name][image-name]`. I also did `docker ps` and noticed that I had no properly defined ports, so I gracefully killed my container and other containers that were running on 8080:80 ports. My container was to run on 8080:80 ports but two containers can't share one port. Even though I ran the same app, and I mapped Terraform and Docker to different Host Ports (8080 and 8091), they do not fight for the same entrance to my computer. As long as they both point to Internal Port 80 inside their own boxes, they will both serve my webpage perfectly. When I was done, I still did the process of `docker build` - `docker push` - `terraform init` - `terraform plan` - `terraform apply` and the css still didn’t display!!!!

- Error 7: I forgot to update my docker image to a new version!!! Because, Docker images are immutable (they cannot be changed once they are built), you must build a new version (like v2) to update your changes and replace the old v1 container.    So, I did just that, I built and ran a new container with my chosen ports and the web image v2, then pushed the container. I also remembered to edit that change in my terraform/main.tf file under the resource name. Then I did the process again; `docker build` - `docker push` - `terraform init` - `terraform plan` - `terraform apply` and it worked!!!