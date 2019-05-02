[Index](index) > Kubernetes and container orchestration
=======================================================
_In this section, we'll create our own Kubernetes cluster on Azure, and deploy speedtest-logger, speedtest-api and speedtest-web._

Creating the cluster
--------------------
TODO: Here we create the cluster. We end up with visiting k8s dashboard.

Deploying speedtest-api as a service
------------------------------------
TODO: Here we introduce Kubernetes YAML-files, and write a deployment and service for speedtest-api.

Running speedtest-logger as a job
---------------------------------
TODO: Less introduction, more faster. At this point we probably inspect the logs.

Deploying speedtest-web
-----------------------
TODO: Very fast, just the same as speedtest-api.

What now?
---------
We have deployed the basic speedtest system to Kubernetes, but what should we do with speedtest-scheduler? Join us in the [next section](4-third-party-queue) where we'll install a third-party queue, and trigger speedtest-logger based on messages from speedtest-scheduler.