Development workflow  
![development workflow](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/fulldevelopmentworkflow.png?raw=true)

Run image from gcr  
```text
docker run -p 9000:8080 gcr.io/{image_name}
```  
The -p flag tells Docker that requests to a port on 9000 should be forwarded to a port on the container (8080).

### Inside a container image

A container image contains one or more programs and all the files those programs need in order to run.  
![inside container image](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/insideacontainerimage.png?raw=true)

The second part of the container image is the image configuration:  
- Working directory.  
  If the process opens a file using a relative path, the kernel uses the process working directory to figure out the absolute path.
- Environment variable.  
  Used to pass configuration to a process.
- User ID (the default is to run as root).  
  With the user ID, the kernel applies access control (what files the process can read or write to).

### The Linux Kernel

The kernel is the central piece of software that controls access to system resources. It schedules processes, controls networking, manages memory, and provisions a filesystem.  
If a process wants to do something only the kernel can do, it sends a request using the system call API; these requests are commonly called `syscalls`, via a higher-level library.  
![linux kernel](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/linuxkernel.png?raw=true)

The kernel controls almost everything a process can do, see, and use. They can isolate a process by changing its perception of its environment. A process in a container will see a virtual ethernet interface with a local IP.  
![isolated container](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/isolatedcontainer.png?raw=true)

### What happen when we start a container?

When we start a container, Docker works with the kernel to start a program in the container image.  
It clones the files in the container image, which become the root directory of the new process.  
The process can only see and interact with child processes (processes that it starts).  
The kernel will make the process believe it has its own hostname and network stack with a local IP address.

### How to build a container with Docker?

Docker takes local files (Docker calls this the "build context") and a Dockerfile and turns them into a container image using `docker build` command.  
Docker starts by sending the build context (the local directory) to Docker. It then starts to execute the Dockerfile line by line.    
```text
Sending build context to Docker daemon  88.06kB
Step 1/7 : FROM golang:1.15
1.15: Pulling from library/golang
627b765e08d1: Pull complete 
c040670e5e55: Pull complete 
073a180f4992: Pull complete 
bf76209566d0: Pull complete 
6182a456504b: Pull complete 
73e3d3d88c3c: Pull complete 
5946d17734ce: Pull complete 
Digest: sha256:ea080cc817b02a946461d42c02891bf750e3916c52f7ea8187bccde8f312b59f
Status: Downloaded newer image for golang:1.15
 ---> 40349a2425ef
Step 2/7 : WORKDIR /src
 ---> Running in 72dd25a0e38d
Removing intermediate container 72dd25a0e38d
 ---> 85f22e89df14
Step 3/7 : COPY go.* ./
 ---> daf048202ca0
Step 4/7 : RUN go mod download
 ---> Running in 7fd6bf604a74
Removing intermediate container 7fd6bf604a74
 ---> 46159d6bc98f
Step 5/7 : COPY . /src
 ---> 88e1af01578b
Step 6/7 : RUN go build -o /main
 ---> Running in 03146e3b49a9
Removing intermediate container 03146e3b49a9
 ---> b91939a3b6b8
Step 7/7 : ENTRYPOINT ["/main"]
 ---> Running in f84e18bc58b7
Removing intermediate container f84e18bc58b7
 ---> 5345d20f192d
Successfully built 5345d20f192d
Successfully tagged hello-go:latest
```

### Docker file instructions

| Instruction  | Files                                                                         | Image Configuration                   |
|--------------|-------------------------------------------------------------------------------|---------------------------------------|
| `FROM`       | Overwrites files                                                              | Overwrites configuration              |
| `COPY`       | Adds files from the build context (local files) or a named stage to the image | Not applicable                        |
| `RUN`        | Runs a command in the image and saves the changes made to file                | Not applicable                        |
| `WORKDIR`    | Creates the directory if it doesn't exist                                     | Changes the default working directory |
| `ENV`        | Not applicable                                                                | Adds an environment variable          |
| `ENTRYPOINT` | Not applicable                                                                | Changes the default command to run    |

We can use package manager to install additional tools.

If we want to deploy a container image to production, smaller is better!  
Security is an additional reason to have small container images in production that only contain what is needed. The vulnerabilities come from libraries that included in our image.  
The [Distroless](https://github.com/GoogleContainerTools/distroless) project aims to deliver a selection of container images that contain just enough dependencies to run an application, and nothing more.  
```text
REPOSITORY                        TAG       IMAGE ID       CREATED          SIZE
hello-go-multistage               latest    a839e685194d   7 seconds ago    25.7MB
hello-go                          latest    5345d20f192d   4 minutes ago    2.1GB
```

### Artifact Registry

This is successor of Container Registry. It helps us host and distribute container images.  
Enable Artifact Registry  
```text
gcloud services enable artifactregistry.googleapis.com
```  
How to create a repository  
```text
gcloud artifacts repositories create cloud-run-book \
  --location=asia-southeast1 \
  --repository-format=docker
```

#### How to build, push and deploy the image into Cloud Run?

Set up another environment variable with your project ID  
```text
PROJECT=$(gcloud config get-value project)
```  
Construct the image URL in another environment variable  
```text
IMAGE=asia-southeast1-docker.pkg.dev/$PROJECT/cloud-run-book/hello
```  
Build and tag the container image  
```text
docker build . -t $IMAGE -f Dockerfile
```  
Set up credential  
```text
gcloud auth configure-docker asia-southeast1-docker.pkg.dev
```  
Push the image to Artifact Registry  
```text
docker push $IMAGE
```  
Deploy the image to Cloud Run  
```text
gcloud run deploy hello-world \
  --image $IMAGE \
  --allow-unauthenticated
```

### Can we build a container without docker?

For Java, we can use [Jib](https://github.com/GoogleContainerTools/jib). If we add the plug-in, building a container is as simple as typing mvn compile `jib:dockerBuild`. The resulting container is built on top of the Java Distroless container.

Another tools are [Cloud Native Buildpacks](https://buildpacks.io/). Buildpacks turn source code into a container image, just like Jib. The difference is that Buildpacks are standardized—anyone can create a Buildpack to turn any source code into a container image.

#### Cloud Build

Google Cloud also offers a service to build container images remotely on Google Cloud: Cloud Build.  
Enable Cloud Build on our project  
```text
gcloud services enable cloudbuild.googleapis.com
```  
Inside our repository  
```text
PROJECT=$(gcloud config get-value project)
REPO=asia-southeast1-docker.pkg.dev/$PROJECT/cloud-run-book

gcloud builds submit --tag $REPO/hello-cloud-build
```  
This command first sends our local directory to Cloud Build, and then executes docker build using a virtual machine on Google Cloud. The resulting container image is then sent to Container Registry. We can deploy with  
```text
gcloud run deploy hello-cloud-build \
  --image $REPO/hello-cloud-build \
  --allow-unauthenticated
```

![cloud build](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/cloudbuild.png?raw=true)