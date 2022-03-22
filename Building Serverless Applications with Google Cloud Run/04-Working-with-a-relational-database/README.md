Enable Cloud SQL in our Google Cloud project  
```text
gcloud services enable sqladmin.googleapis.com
```  
This command enables the Cloud SQL Admin API.

Create the Cloud SQL instance  
```text
gcloud sql instances create {instanceName} \
--cpu={numberCPUs} \
--memory={memorySize} \
--region={region}
```  
or  
```text
gcloud sql instances create {instanceName} \
--tier={apiTierString} \
--region={region}
```

## How we connect to Cloud SQL

1. [Cloud SQL Proxy](https://cloud.google.com/sql/docs/mysql/sql-proxy).  
2. Direct connection.

### Cloud SQL Proxy

![cloud proxy](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/cloudsqlproxy.png?raw=true)

Cloud SQL Proxy will automatically set up a secure SSL/TLS connection to the Cloud SQL Proxy Server, which runs on the Cloud SQL instances next to the database server.  
The Cloud SQL Proxy Server authenticates incoming connections using Cloud IAM.

How to connect  
```text
./cloud_sql_proxy -instances=INSTANCE_CONNECTION_NAME=tcp:3306
```  
The proxy will listen to port 8080  
```text
Listening on 127.0.0.1:3306 for INSTANCE_CONNECTION_NAME
Ready for new connections
```  
We can use any mysql client to connect 127.0.0.1:3306

We can connect via gcloud using  
```text
gcloud sql databases create todo --instance {instanceId}
```

By default, the MySQL instance has a superuser, root, without a password, and any host (%) can connect. How to secure it?  
1. Delete user root.  
```text
gcloud sql users delete root --host % --instance {instanceId}
```  
2. Create the user root again, but they can only log in through the Cloud SQL Proxy. The Cloud SQL Proxy handles authentication for us.  
```text
gcloud sql users create root --host "cloudsqlproxy~%" --instance {instanceId}
```

How about Cloud Run, how they connect to Cloud SQL?  
We can connect it to Cloud SQL using the flag --add-cloudsql-instances. This will add a special file to your container in the directory /cloudsql/ (UNIX Domain Socket).  
![unix domain socket](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/unixdomainsocket.png?raw=true)

### Direct connection

The direct connection is not encrypted by default. We can require SSL/TLS, but that means we'll need to generate client certificates and get them to our Cloud Run container securely.  
The direct connection will always bypass Cloud IAM, and we'll need to start managing the firewall rules of our Cloud SQL instance.

Use this command to enable SSL/TLS  
```text
gcloud sql instances patch INSTANCE_NAME --require-ssl
```

## Cloud SQL IP

Cloud SQL uses a public IP and protected by the Cloud SQL Proxy server, the built-in firewall, and SSL/TLS with a client certificate (if we require SSL).  
We can choose to assign only a private IP to our Cloud SQL instance. A private IP is accessible only from within our Virtual Private Cloud (VPC) network.  
![private ip](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/privateipwithvpcpeer.png?raw=true)

We can connect Cloud Run to Cloud SQL using --add-cloudsql-instances and the UNIX Domain Socket, protected by IAM, regardless of whether we use a private or public IP.

## Limiting Concurrency

Transaction concurrency is the number of transactions (queries) that the database server is handling at the same time.  
It's good to have low concurrency rather than high. The common heuristic for optimal concurrent value is a small multiple of the number of vCPUs of the machine.

### The effect if we have high concurrent value

![concurrent vs rate](https://github.com/bluething/learn-googlecloudrun/blob/main/Building%20Serverless%20Applications%20with%20Google%20Cloud%20Run/images/concurrecnyvsrate.png?raw=true)  
Transaction rate is expressed as transactions completed per second (TPS).

The reason why transaction rate start decrease at certain point is resource contention.  
To solve contention the system need more time, increasing the duration of transactions.  
The common cases for contention is lock.

#### How we solve this?

1. Limits the maximum number of containers Cloud Run will add.  
```text
gcloud run services update {serviceName} --max-instances 100
```  
2. Use a connection pool.  
3. Use external connection pool such as PgBouncer, on a Compute Engine virtual machine in front of our Cloud SQL instance. 