Three ways to interact with Google Cloud:  
1. The command-line interface gcloud. How to [install](https://cloud.google.com/sdk/docs/install)  
2. The web console at console.cloud.google.com.  
3. The Cloud Console mobile app.

On Google Cloud, a project is how we organize your applications. Every cloud resource we create has to belong to a single project.  
Resources in the same project are aware of one another and can communicate, but resources in different projects are isolated from one another.

Deploy  
```text
gcloud run deploy {serviceName} \
--image gcr.io/{containerName} \
--allow-unauthenticated
```  
The `--allow-unauthenticated` flag in the deploy command ensures us can access the URL without passing an authentication header.

Deploying a new version  
```text
gcloud run services update {serviceName} \
  --image gcr.io/{containerName}
```

Every revision has an immutable copy of the service configuration:  
1. The full contents of the container image.  
2. Container configuration: arguments, overriding the default program, and environment variables.  
3. Serving configuration, including request timeouts and the default port.  
4. Resource limits: CPU and memory allocation.  
5. Scaling boundaries: minimum and maximum instances.  
6. Platform configuration: service identity and attached resources (network, database).

We can list the revisions Cloud Run has created with this command  
```text
gcloud run revisions list --service {serviceName}
```

![revision configuration](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/revisionconfiguration.png?raw=true)

Revisions are kept unless we delete them, which means we can switch back to an earlier revision with one command. For example, we roll back 100% of traffic to other revision 
```text
gcloud run services update-traffic {serviceName} \
  --to-revisions {revisionId}=100
```

Structure of the HTTPS Endpoint  
```text
https:// [name] - [project hash] - [region hash] .a.run.app
```

Container life cycle  
![container life cylce](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/containerlifecycle.png?raw=true)  
When SIGTERM send by Cloud Run we need to prepare shutdown programmatically, especially for database.

Cloud Run guarantees full availability of the CPU as long as a container is handling HTTP requests. If a container is not handling requests, our container will still get a slice of CPU time occasional, but it's not enough to do meaningful work.  
What about task scheduling? For example photo uploading, we don't want user wait until all processes (scale, crop, and recompress) finished.

How load balancer and autoscaling work  
![load balancer and autoscaling](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/loadbalancerandautoscaling.png?raw=true)  
Load balancer uses a concurrent request limit to decide if a container can accept another request.  
```text
gcloud run deploy {serviceName} \
  --image [IMAGE-URL] \
  --concurrency {concurrencyLimit}
```  
If there are no free request slots available on any container, an incoming request is temporarily held in the request buffer until a slot frees up. If there are no (zero) containers available, the request waits until a new container is ready.

Autoscaler work by reading metrics from container to determine the number of containers that should be available to handle the requests.  
They also read CPU utilization. If CPU usage on the containers is high, the autoscaler can decide to add a container even when there are still enough request slots available.  
The autoscaler will keep adding containers if necessary until it reaches the maximum limit. If exceed they will return requests with the HTTP 429 error status.

If our application have traffic burst behavior, we need to set concurrency with right value to make autoscaler react on request slots rather than CPU utilization.

Cold start is state when our application up until ready to listen 8080 port.  
It happens when:  
1. Creating new revision.  
2. Scaling up.  
3. Our service doesn't receive traffic for a while.

We can _set minimum instances_ that tells the autoscaler to always keep a certain number of containers ready.

Disposable is container state when:  
1. Reboot.  
2. Become unhealthy.  
3. Disappear due to scaling.

The disposable containers in Cloud Run have a small disk (RAM). This memory share with the app.

Cloud Run sends requests to a container as soon as it accepts new connections on port 8080. They don't care if the app ready or not.  
If a container on Cloud Run sends more than 20 consecutive responses with HTTP status 5xx (500 to 599), the container is taken out of service and replaced.
