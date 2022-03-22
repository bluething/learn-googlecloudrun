Using Cloud Run doesn't mean we lock in to Google Cloud because:  
1. Our application must be packaged in a container, portable.  
2. Cloud Run platform is based on the open Knative specification, which means we can migrate your applications to another vendor or hardware.

What people think about serverless  
1. We only focus on the code and give architecture things to the platform.  
2. Autoscaling based on demand.  
3. We pay for actual usage only, not for the pre-allocation of capacity. Serverless != cheap, it's happen when we utilize close to 100% of our server capacity all the time.  
4. Serverless not always Function as Service.

In Google Cloud there are no serverless relational database.

Cloud Run developer workflow:  
1. App that listen to HTTP port.  
2. Package into a container image.  
3. Deploy the image to Cloud Run.

The architecture of Cloud Run is designed to be limited only by the available capacity in a given Google Cloud region (a physical datacenter).

By default, Cloud Run automatically creates a unique HTTPS endpoint we can use to reach our container.

Every Cloud Run service has an assigned identity to make sure that every Cloud Run service only has the permissions to do what it is supposed to do (principle of least privilege) and that the service can only be invoked by the identities that are supposed to invoke it.

Cloud Run captures standard container and request metrics. Use structured logs with appropriate metadata to debug issues in production. Our application logs are forwarded to Cloud Logging.

Concerns about serverless:  
1. Unpredictable costs. We can set boundaries to scaling behavior and tune the amount of resources to mitigate this.  
2. Hyper-Scalability. Watch out for the downstream, maybe can't scale as upstream.  
3. When you run your software on top of a platform you do not own and control, you are at the mercy of your provider when things go really wrong.  
4. Latency. If we use Cloud Run, we need to store data that needs to be persisted externally in a database or on blob storage. We depend on how fast the network.  
5. Open source compatibility. How easy is it for us to migrate our application from one vendor to another.