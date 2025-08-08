---
title: "From Dockerfile to Distroless - Container Lab"
description: "Getting your container basics right"
difficulty: "easy"
duration: "50 min"
topics: ["Containers", "Docker", "DevOps"]
logo: "/assets/logos/docker-logo.png"
resources:
  - title: "Lab Repository"
    url: "https://github.com/tsclabs-eu/lab-container-basics"
  - title: "Docker Documentation"
    url: "https://docs.docker.com"
  - title: "Twelve Factor App"
    url: "https://12factor.net"
---

## Overview

<img src="/assets/img/boat-security.webp" style="width: 40%;" />

<br />

In this Lab, we'll start with a very basic app, create a container, and improve it iteratively. 

During this Lab, you will complete the following tasks:
* Inspect an app and apply Twelve-Factor principles
* Building a simple container
* Switch to a {{distroless}} image
* Run as a non-root user
* Sign and validate signatures

While doing this, we will take a closer look at code style and performance of the container build process.


## Pre-requisites
To work through this lab, you will need the following pre-requisites, either on your machine or on a cloud-based virtual machine:

* A POSIX-compatible shell (bash, zsh). Windows Subsystem for Linux (WSL) should work fine
* Git CLI installed
* A container engine installed (Docker, Rancher Desktop, or colima)
* Throughout this lab, you will also need docker compose, which is typically part of your Docker installation
* A GitHub or Docker Hub account (optional, for pushing your changes)

:::warning System Requirements
Make sure that Docker is running before starting this lab.
:::

This should be enough to start with this Lab.

## Checking out the example app
Imagine your boss with an affinity for {{vibe-coding}} wrote a very sophisticated app and handed it to you in a Git repository. The app is written in JavaScript, provides a sleek frontend to track your learning goals and progress, and stores data in a {{SQLite}} database.

:::tip Pro Tip
Feel free to fork the repository if you want to save your changes and experiment with different approaches.
:::

To check out the application, create a playground directory somewhere on your machine and clone the repository:

```
git clone https://github.com/tsclabs-eu/lab-container-basics
```

Afterward, you should see a new directory on your machine containing the application files.

This will be the starting point for your container journey. There is an `app` directory inside of this repository, which contains the application code. 

:::info Feedback
If you find any issues with the lab or have some feedback, please open an issue or create a pull request in this repository.
:::

## Writing your first Dockerfile

Containers have been around for a long time. In the early 2010s, Docker made it easy to build recipes to create containers. These recipes are called Dockerfiles and even if you're not using Docker as a Container Engine or Runtime, it's a common convention to call such files Dockerfiles.

Now it's time to open your favorite IDE and create a file named `Dockerfile` in the root of the checked-out repository.

### Base Images
One of the major benefits of containers is the reusability and efficiency in terms of disk space usage. This is achieved by using a layered filesystem, which you can imagine as a series of zip files, extracted over each other.

In our case, we will take a pre-created {{container image}} representing the Linux distribution and some packages needed to run the Node.js application in our container. This one is stored in a registry (Docker Hub) and we can simply download it.

:::info Info
Although we don't need it in our Lab here, you can always pull an image using the command `docker pull <image-name>` to download an image into the cache of your machine.
:::

In our Dockerfile, we can use such a base image using the `FROM` directive, which will fetch the image from the container registry and let us use files, binaries, and libraries from it. Therefore, please open your favorite editor, open your Dockerfile and add the following to it:

```
FROM debian:bookworm-slim
```

Image names typically follow this format: `registry/namespace/image:tag`. For example:
* `debian:bookworm-slim` (official image from Docker Hub)

* `ghcr.io/tsclabs-eu/lab-container-basics/learning-app:v1.0` (from GitHub Container Registry as we will see later)
* if you omit the tag given, "latest" is used to pull or push the image
* avoid using latest wherever possible. The "latest" tag can change without warning and break reproducibility
* Container engines have their default registries. When using Docker, this is Docker Hub. Other engines might use different registries.


This is the most simplistic Dockerfile you can create. Although it won't do much more than download the base image, it is possible to build it. 

:::note Note
The Dockerfile itself is only a description of the build process and only changing the Dockerfile will do nothing, you have to run a build command to get your container built.
:::

Let's start our first container build, so open your shell, navigate to the directory of your repository and run:

```
docker build -t my-first-container .
```

This command consists of four parts:
* The name of the executable (`docker`)
* A directive on what to do (`build`)
* Options for our build (`-t my-first-container`). This will tell the build process to name the resulting container image `my-first-container`.
* A context (.) that tells the container engine which directory to use as the root of the build. Any local files referenced in the build are looked up relative to this directory.

After executing this command, you should get an output like

```
[+] Building 41.0s (5/5) FINISHED                                                    

[...]

 => exporting to image                                                                        0.0s
 => => exporting layers                                                                       0.0s
 => => writing image sha256:afb9fee037e00356afc763f5b7a40982fd647c2d35282aa6e807bbf14b10c34f
 => => naming to docker.io/library/my-first-container
 ```

 When everything finished successfully, the container should be in your local image cache and you should be able to run it:

```
docker run --name my-first-container my-first-container
```

:::warning Common Issues
If your container doesn't behave as expected:

- **Port already in use?**  
  Use a different local port: `-p 8080:3000`

- **Container won't start again?**  
  Remove it: `docker rm learning-tracker`

- **Not seeing logs?**  
  Use `docker logs learning-tracker` to debug.

- **Unsure what’s running?**  
  Run `docker ps -a` to list all containers.
:::

You might have noticed that the structure of the docker command is very similar to the build command, you still have the same executable, but a different verb and in this case a mandatory path.

If everything goes well, you should be in your shell again. But why? 

Typically, a container runs as long as there is something to do. In the Debian container, there is no long running process in there. Therefore, the command which gets executed (`bash`) finishes, and therefore the container terminates. You can inspect the state of your container using the command `docker ps -a`.

```
CONTAINER ID   IMAGE               COMMAND             STATUS                      PORTS     NAMES
4d23f2b3cd2e   my-first-container  "bash"              Exited (0) 2 minutes ago              my-first-container
```

You can see that the container is in the state "Exited" and therefore not running anymore. If you want to run the container, but keep its shell open, you can use the `-it` parameter to run it in interactive mode:

```
docker run -it --name my-first-container my-first-container
```

Now, you might experience that Docker shows an error that the container name is already in use. This is because the container is still there, but not running. You can remove it using the command:

```
docker rm my-first-container
```

After this, you can run the container again using the command above. Now you should be in a shell inside of the container. You can run commands like `ls -la` or `pwd` to inspect the container.

:::info Additional Information
If you want your container to keep running, ensure that the process you're spawning does not terminate
:::

Next, we’ll switch to a base image designed for running Node.js services.

### Building a Node.js Application

Currently, our container build is not very sophisticated. To create more value, we have to add some more directives to it.

At first, we want to copy files from our repository into the container (keep in mind that the container only has a base image at the beginning, therefore we have to add the files we need). Please open your `Dockerfile` again and replace the contents with the following lines:

```
FROM node:22

COPY app /app
```

This will change the base image to the Node.js-image, but also copy the contents of the local app directory (remember that we use the context .) into the /app directory of the container.

After this, please add the line

```
WORKDIR /app
```

into your Dockerfile. This will ensure that all of the commands you are executing are running in the directory we copied in the container before.

Afterward, we have to download our dependencies. Like in every npm-based JavaScript application, we can achieve this using the command `npm install`, which can be executed in the Dockerfile using

```
RUN npm install
```

The `RUN` directive makes it possible for you to run commands inside of a container. Since the `node` image includes npm, this works out of the box.

Last but not least, we want to execute the application when the container gets started. For this, we can use the `CMD` directive to start the npm process:

```
CMD ["npm", "run", "start"]
```

:::info CMD vs ENTRYPOINT
In Dockerfiles, `CMD` specifies the default command, but it can be overridden at runtime. `ENTRYPOINT` is used when you want to enforce the command and just pass arguments. Most Node.js apps use `CMD`, as you’ve done here.
:::

The resulting Dockerfile should look like this:

```
FROM node:22

COPY app /app

WORKDIR /app

RUN npm install

CMD ["npm", "run", "start"]
```

Furthermore, we want to ensure that build artifacts from local builds will not get copied inside the container. Therefore, we can create a `.dockerignore` file that will avoid copying specified files to the container. We want to do this with the `app/node_modules` folder. Please create a new `.dockerignore` file in the root of your repository and add the following contents:

```
app/node_modules
```

After this, we should be able to build our container. 

```
docker build -t learning-tracker:v0.1 .
```

This will build and create our container. After this has been finished, you should be able to run the container using the command

```
docker run --name learning-tracker --rm learning-tracker:v0.1
```

In this command, the `--rm` option ensures that the container is removed after it has been stopped, so you don't have to clean up manually. The `--name` option gives the container a name, which makes it easier to reference later.

Now you should see some output like this:

```
Server running on http://localhost:3000
```

Now your fancy service is running, but if you're trying to browse http://localhost:3000, you'll find out that you will not be able to access something, or at least not your service. What could be wrong here?

:::tip Important
Think about your container as a process that is put into a box. It can not be accessed from the outside and cannot access data from the outside. Resulting, you have to specify at start if you want to forward ports from your host system to the container and if you want to use data from your host system.
:::

With this in mind, we will forward the application port (in our case 3000) to a local port to access our new application. Therefore, we have to add the parameter `-p <local-port>:<container-port>` to the docker command, similar to:

```
docker run -p 8888:3000 --name learning-tracker --rm learning-tracker:v0.1
```

When navigating to `http://localhost:8888` in your browser, you should now see the shiny learning-tracking application.

:::congratulations Well Done!
You've successfully created your first container! You've learned how to write a Dockerfile, build a container image, and run it with port forwarding. This is a major milestone in your container journey.
:::

## Twelve Factors
Now that we created our first container, we’ll now look at how our application behaves and what might help us make it better and more cloud native. For this, we will take a closer look at the Twelve-Factor App methodology.

The [Twelve Factor App Methodology](https://12factor.net) is a set of criteria which should ensure that applications are portable, scalable and maintainable. They were initially created by Heroku, but are widely accepted as a best practice for building modern applications and open-sourced.

For our example, the following factors are relevant:
* **Logging**: The applications should not write to files, but to {{stdout}}/{{stderr}}. This allows the container engine to collect the logs and forward them to a log collector. Furthermore, this ensures that the logs are not lost when the container is restarted and the filesystem is not flooded with log files.

* **Configuration**: The application should not have any hardcoded configuration, but use environment variables (or an external configuration file) to configure the application. This allows the application to be configured at runtime and makes it easier to deploy the application in different environments.

* **Backing Services**: We will ensure that the database used by the application is not hardcoded, but can be configured via environment variables. Furthermore, we will ensure that the application and the database are not tightly coupled, but can be deployed and scaled independently.

At first, let's take a closer look at our application and find out how it's logging. One indicator that the service is not logging too much is that it is very quiet at the moment. If you're accessing some endpoints, you don't get any response on your shell and there is not more log output around it. Therefore, we can open up a shell in the container and inspect it a bit. Please start the container again (now in detached mode using the `-d` flag). Open a new shell and run the commands

```
docker run -d -p 8888:3000 --name learning-tracker --rm learning-tracker:v0.1
docker exec -it learning-tracker /bin/bash
```

With this, you should have a shell opened in your container. The easiest thing you could do now is to execute a simple `ls -la` to find out which files are stored in the current directory (which was `/app` in our case). You will find out that there is a file called `app.log` in the directory. Which problems could we have with this?

:::warning Warning
Writing logs to the container filesystem could cause multiple issues. At first, it could cause the filesystem to run full and therefore put the stability of the container at risk. Secondly, container filesystems are ephemeral and logs will be gone when your container gets restarted. Last but not least, you should always aim to run the root filesystem of your container read-only. The logs which are written there, will make this effort a bit more complicated.

At this point, we found out that we're logging to the local filesystem, but how could we change this behavior? Typically, there's an option in the used programming language or framework to change this to stdout/stderr. 

In our case, we can inspect the app a bit and if we take a closer look in the `app.js` file, we'll find out that there is an option to also write log messages to the console by changing an environment variable (line 19). Fortunately, we have the option to pass over environment variables by adding the `-e` parameter with the variable and its value to the run command of the container. 

At first, we need to stop the currently running container, as we started it in detached mode. You can do this using the command:

```
docker stop learning-tracker
```

Using `docker ps` you can check that the container is not running anymore. After this, we can start the container again, but this time with the environment variable set to `console`:

```
docker run -e LOG_OUTPUT=console --rm --name learning-tracker learning-tracker:v0.1
```

When starting this, you will experience that the container will now log directly to stdout. 

:::tip Logging done!
At this point, you fixed logging in your application and your container will not log into the filesystem anymore
:::

Initially, we discussed that this application is using SQLite as a backend, which is by definition a file-based database. This is a pretty cool thing as long as we're running on the local machine, but as soon as we want to scale and maybe run multiple application servers this will become a problem. The database is not replicated and therefore every instance of the application will have different data which is suboptimal in a world where we don't know which backend we'll arrive next.

Therefore, we have two options: Switch to an external database or ensure that our container is only running once. The second one should not be an option nowadays.

In the next step, we'll externalize the database and learn about how to orchestrate multiple containers on a single host.

## Orchestrating multiple containers

So far, we only had one container running on our machine. In the next step, we'd like to have our application container and a database running there. Our Boss had this in mind and created an option to switch between a SQLite database and a MariaDB as a database backend. All we need is a good approach to make this work. 

A good rule of thumb is to run one application or service per container so each can scale independently. In our case, that means running the database in a separate container from the application.

To manage multiple containers together, we’ll use Docker Compose, a simple tool for defining and running multi-container applications.

Modern Docker installations include Docker Compose v2, which you run with a space:

```
docker compose 
```

Older installations may still use the legacy standalone binary, which uses a hyphen:

```
docker-compose
```

:::tip Quick Self Check
Before continuing, run:

```
docker compose version || docker-compose version
```

* If the first command works → you have Compose v2, and can follow the lab exactly as written.

* If only the second works → replace `docker compose` with `docker-compose` in all commands.

* If neither works → follow the [official installation guide](https://docs.docker.com/compose/install/) before proceeding.
:::

Docker Compose will help us to create deployments consisting of multiple containers, so that we could run our application together with a database container. The tool is typically configured using a `docker-compose.yaml` file that describes our application. Let's start very small and put our current app into a docker-compose.yaml file. Please open your favorite IDE and create a file called `docker-compose.yaml` in the root of your checked out repository. Add the following configuration in it: 

```
services:
  learning-app:
    container_name: learning-app
    build: .
    environment:
      LOG_OUTPUT: "console"
    ports:
      - "8888:3000"
```

If you take a closer look at it, this configuration has everything we put into our docker command before, but in a more declarative format. After we created this file, we can run this docker-compose setup using the command:

```
docker compose up
```

When executing this one, you should see the log output and be able to navigate to the app via `http://localhost:8888`. You can stop this using Ctrl+C.

We would like to change the database to a MariaDB that is running alongside our learning-app. To achieve this, we need to run the database in a second container. To add this container, please change your docker-compose.yaml file that it matches this configuration:

```
services:
  learning-app:
    build: .
    restart: always
    environment:
      LOG_OUTPUT: "console"
      DB_TYPE: "mysql"
      DB_USER: "app"
      DB_PASSWORD: "my_password"
      DB_HOST: "mariadb"
    ports:
      - "8888:3000"
  mariadb:
    image: mariadb
    environment:
      MARIADB_USER: "app"
      MARIADB_RANDOM_ROOT_PASSWORD: "yes"
      MARIADB_PASSWORD: "my_password"
      MARIADB_DATABASE: "learning.db"
```

There are lots of new things inside of this configuration, let's take a closer look at them.

At first, we created a new MariaDB service to the configuration, which will start our database. According to the [documentation](https://hub.docker.com/_/mariadb), the container image takes a MARIADB_USER, MARIADB_PASSWORD, and a MARIADB_DATABASE as environments for configuration. In our app.js, the default name for the database defaults to "learning.db", our very secure password is my_password. Furthermore, defined services in a docker compose file are accessible with their service-name as a hostname (in our case this is mariadb), which is added as an environment variable in our learning-app. Furthermore, we can be sure that the database takes longer to start up as our application, as our application is dependent on our database, we would have the option to wait until the health check of the database is successful (which we will not do in this lab), or restart until everything is ok (which matches the behavior of kubernetes).

:::info Additional Learning
If you are curious, try to find out how to define a health check in Compose and use and work with the `depends_on` in the `docker-compose.yaml`
:::

Before starting this, ensure that the old deployment is stopped (Ctrl-C) and also delete the deployment including all created volumes using the command:

```
docker compose down -v
```

Afterward, start the new deployment:

```
docker compose up
```

You should see much log output, and in the end a message that your service is listening on port 3000, which is mapped to the local port 8888 in our case. When browsing to this page, you should see our learning-tracker again and be able to add new goals and move them through the columns.

:::congratulations You finished the next milestone!
By now, you have a pretty solid setup of a container logging to stdout, using an external database. By configuring the containers via environment variables, you are also respecting the configuration factor of the Twelve-Factor App. 

Well done!
:::

## Reducing the attack surface
At the moment, we have a good starting point for our Node.js application, but there's still room for improvement. Let's investigate a bit. Simply start your docker-compose setup and open a shell to it (if it's not running):

```
docker compose up -d
docker exec -it lab-container-basics-learning-app-1 /bin/bash
```

You learned a new thing here, so far your compose setup was running in the foreground and when you hit Ctrl+C it got removed. By using the `-d` flag (for detached), everything gets started in the background. Therefore you can continue working in the same shell.

After you executed the `docker exec` command, you are inside the shell of our learning app container. The prompt should look similar to this:

```
root@427b81298043:/app
```

In fact, this prompt alone shows us two things we could avoid in production (when using Node.js):

* The container runs as root
* The container has a shell

In the following steps, we will try to avoid this and therefore reduce the attack surface of the container by utilizing multi-stage builds and using a {{distroless}} image.

### Multi-Stage Builds
Multi-Stage builds can help us separate the steps to build the software inside a container from the container which is really running in production. In our case, we will spin up a Node.js image, which takes over all of the build steps (therefore downloading dependencies). In a second step, we will run a {{distroless}} container, copy the app directory in it and simply run our app there. Let's change our `Dockerfile` to reflect these changes:

```
FROM node:22 AS builder

COPY app /app

WORKDIR /app

RUN npm install

# Our Runtime Container
FROM gcr.io/distroless/nodejs22-debian12

COPY --from=builder /app /app

USER nonroot

CMD [ "/app/app.js" ]

```

After this, you can stop the docker compose setup, rebuild the containers and restart it again:

```
docker compose stop
docker compose build --no-cache
docker compose up
```

Now, the container starts again and you should be able to browse to it again. Everything should look as before, with some minor differences. Try to open a shell inside of the container:

```
docker exec -it lab-container-basics-learning-app-1 /bin/bash
```

Now you should see a message that the command `bash` cannot be found. This should be the same if you use `/bin/sh` or any other shell. Furthermore, our process is now executed with the user nonroot, which is baked into the {{distroless}} image we are using for our production container. 

:::congratulations
With this, our attack surface is reduced, the only downside is that we cannot spawn a shell into our container which can be changed by switching to a different image for debugging purposes. This should be no problem, if your application exposes as much telemetry data (metrics and traces), to get a clear idea on what's going on
:::

## Improving caching
Our `Dockerfile` is pretty solid in the meantime, but as in every build process, time matters and we could improve a minor thing. Docker is able to cache already done steps and therefore it tries to find out if something has changed during the last build. This is done layer by layer. Currently, it would rebuild everything when something in our `Dockerfile` changes. 

As our build process mainly consists of downloading dependencies, we could improve this a bit in a way that we first only copy the files that define which dependencies are used (package.json and package.lock), run the download process and afterward copy the rest to the container. Let's change our `Dockerfile` accordingly:

```
FROM node:22 AS builder

RUN mkdir /app

COPY /app/package*.json /app

WORKDIR /app

RUN npm install

COPY /app /app

# Our Runtime Container
FROM gcr.io/distroless/nodejs22-debian12

COPY --from=builder /app /app

USER nonroot

CMD [ "/app/app.js" ]
```

With these minor changes, the `npm install` command should only run when nothing is in the cache or something in the dependency configuration changed.

Feel free to rebuild the container and restart your deployment:

```
docker compose stop
docker compose build --no-cache
docker compose up
```

:::congratulations
You optimized the build process a bit. Although this might not have a big impact in this lab, such optimizations can save you lots of time in real-world scenarios
:::


## Pushing an image to a container registry
At this point, our containers are only stored on our machine. To distribute our containers and make them usable from other machines or platforms, it is necessary to push them to a container-registry. Think about such a registry as an artifact-store where you would store your jar files, npm packages or similar with the only difference that this is for containers.

There are many container registries out there, two very prominent examples are Docker Hub (https://hub.docker.com) or the GitHub Container Registry (ghcr).

Before you can push something to the container registries, you have to log in on the corresponding registry at the command line. Please refer to the documentation of your container registry to find out how to log in. 

* For GitHub: [Working with GHCR](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
* For Docker Hub: [Docker Login](https://docs.docker.com/reference/cli/docker/login/)

To be able to push a container to the container registry, you have to tag it properly. You'll need to tag the image appropriately for your registry. For example, with GHCR:

```
docker build -t ghcr.io/tsclabs-eu/lab-container-basics/learning-app:v1.0 .
```

This command should look pretty familiar to you and the build command should run as we saw it in the rest of the lab. After this has been finished, you can push the image to the repository using the command:

```
docker push ghcr.io/tsclabs-eu/lab-container-basics/learning-app:v1.0
```

:::warning Troubleshooting
If you get an authentication error, check you’re logged in with:

```
docker login ghcr.io or

docker login docker.io
```
:::

This command pushes the container to the registry, so it should be usable from there. Take a look at the User Interface of your container registry to find out if everything worked.

## Sign containers / Verify signatures
In the last step of today's journey, we want to ensure that other people can validate if a container image has really been created by us. For this, we will use the project cosign, which can help us creating a signature, but also validate it afterward. 

At first, you have to create the cosign cli. Please refer to the [Cosign Documentation](https://docs.sigstore.dev/cosign/system_config/installation/) to find out how to install this on your machine.

:::info
You can check if your cosign installation is working with the command `cosign version`.
:::

There are some ways on how to manage keys for cosign. In our case, we will use the most simple way and use a file-based keypair. In cloud-provider environments, you could also use their Key Management Services.

The first step to achieve this is to generate a keypair using the command `cosign generate-key-pair`. This will create a private and a public key.

After this, you can simply sign a container using the command:

```
cosign sign --key cosign.key <your-image-name>
```

This signs the image and writes the signature to your container registry (as an {{OCI}} Artifact).

To verify signatures of containers, you can share the public key and use the following command to verify the signatures:

```
cosign verify --key cosign.pub <your-image-path>
```

You should get a message confirming the signature is valid.

:::tip About this section
There are many ways on how to sign containers and things you can do with cosign. This section's main goal was to give you an idea on how easy signing and validation could be and encourage you to sign your images. You can find more information about this interesting topic in the [Cosign Documentation](https://docs.sigstore.dev/cosign/signing/overview/)
:::

## What you've learned

:::congratulations
You finished this lab about container basics and on how to build solid containers.
:::

Throughout this lab, you worked through the following topics:
* Building a container with `docker build`
* Running a container using `docker run`
* Fundamentals about Twelve Factor Apps
* How to use Docker Compose to run Multi-Container Setups
* Reducing the attack vectors to containers
* Pushing containers
* Sign containers

## What's Next?

Here are some ideas if you want to keep learning:

- Add a health check to your app and `Dockerfile`
- Use Docker Compose volumes to persist database data
- Try using `trivy` to scan your image for vulnerabilities
- Read more about Kubernetes and Helm for production deployment

:::info
I hope you enjoyed this Lab! If you feel like this could be useful to some of your friends or colleagues, please share it. Furthermore, please star the GitHub Repository of this Lab and follow TSC Labs on Linkedin
:::
