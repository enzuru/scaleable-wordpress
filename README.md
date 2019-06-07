# Scaleable WordPress

## Installation

Given that you are pointing to an appropriate *regional* cluster in GCP, installation should be straightforward:

### MySQL
```bash
kubectl apply -f mysql/mysql-configmap.yaml
kubectl apply -f mysql/mysql-services.yaml
kubectl apply -f mysql/mysql-statefulset.yaml
mysql/setup
```

### WordPress
```bash
kubectl apply -f wordpress/wordpress-volumeclaim.yaml
kubectl apply -f wordpress/wordpress-deployment.yaml
kubectl apply -f wordpress/wordpress-service.yaml
wordpress/setup
```
### Prometheus

I setup Prometheus easily with this. If I had more time, I would have liked to have it in the same format as the above MySQL and WordPress resources:

https://github.com/GoogleCloudPlatform/click-to-deploy/blob/master/k8s/prometheus/README.md

## Initial thoughts

I chose to do this with Kubernetes because of strong GCP support and how easy it makes infrastructure, scaling, self-healing, and monitoring. This lays the foundation for a "set it and forget it" model.

### Infrastructure

I only had enough time to setup `us-west2` in all zones, which I believe is the most appropriate region for San Francisco. However, all of this code can be used to create a `asia-northeast1` architecture for Tokyo without any changes.

Ideally, we have nodes in every zone in both `us-west2` and `asia-northeast1`, which gives us incredible stability even if an entire region goes out.

If I wasn't doing this on GKE, maybe I'd do it in Terraform. I'd describe some WordPress and MySQL infrastructure and setup autoscaling groups.

### Scaling

Autoscalers for both WordPress and MySQL slaves; the cluster was also setup with autoscaling. That should automate away most of our concerns, regardless of the numbers being thrown at us. If I had more time, I'd verify some assumptions using Locust.

We need to have MySQL with a single master (for writing) and multiple slave databases (for reading) per region. WordPress needs a plugin called "HyperDB" to accomplish separate master and slave sources; I was initially going to extend a Docker image to do this, but it turns out `bitnami/wordpress:latest` already has it.

I setup a generic "30% CPU" autoscaler. I'm not sure this is completely right for LAMP -- it's very possible queries-per-second would be a better benchmark. I also put artificial maximums on the scalers that wouldn't exist if we were in production.

One of my few LAMP jobs was when I worked at Nordstrom; we did flash sales which meant giant jumps in active users during short time periods. We used to manually scale up and down for those events -- I think we could have done better.

### Self-healing

The WordPress livenessProbe checks the health through a simple `GET /`.

The MySQL livenessProbe checks the health through a `mysql ping`.

### Monitoring

Other than adding StackDriver and MySQL as sources, I didn't get to do too much with Prometheus after setting it up since time was short. I'd be interested in monitoring StackDriver information about the cluster, as well as looking at MySQL's `performance-schema`.

What I'd be looking for in monitoring:

- What are the median and average response times for a page?
- How many critical errors and how often?
- What does CPU/RAM usage looking like across nodes?
- Any slow queries?
- Are we scaling up quickly and gracefully enough under load?

### References

I always try to find initial templates for the software I am building before making my modifications, whether from previous work I've done or reputable outside sources.

Luckily, both Kubernetes and GCP's websites had some pretty fleshed out templates to start from:

MySQL: https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/
WordPress: https://cloud.google.com/kubernetes-engine/docs/tutorials/persistent-disk

# Follow up

Steps I didn't have time for:

- Review and lockdown security. Glaring security flaws here ranging from using default roles to LAMP configuration. No MySQL password!
- Run Locust in a cluster to get a better sense for how autoscaling is working; is CPU-driven scaling really correct for LAMP? Perhaps queries-per-second would be better?
- Review resource management, prioritization, and namespaces for pods
- Create different environments for staging, production, etc with some smoke tests where we could automate updating WordPress and MySQL to the latest Docker images (this practice is big at Netflix!)
- Move Prometheus setup code into repo -- right now we're using click-to-deploy and are manually adding datasources.
- Verify if WordPress is being appropriate about MySQL's master/slave relationships
