# [Index](index) > Helm, the package manager for Kubernetes

_In this section, we'll introduce Helm, and use it to install NGINX, so we better can control access to the services on our cluster._

## Motivation

In the previous sections, we deployed our Kubernetes app using the kubectl command line application. This was painful because we had to remember to create the deployment and the service each time we wanted to release a new version of our app. If we created any more Kubernetes resources, then we'd have to remember to manually deploy those each time too.

On top of that, we also had to remember to update the dependencies between each component/resource in our Kubernetes manifest files. Like the name of the services, and pods. If we've built some automation around this, then we'd need to update our scripts each time we made any changes to filenames.

The problem is that we have to remember exactly how to deploy the application step by step. Our "application" (i.e, all of our Kubernetes resources packaged together) is something kubectl has no idea about.

## Helm

[Helm](https://github.com/kubernetes/helm) is one of the solutions to this problem. According to the documentation:

> [Helm is a tool for managing Kubernetes charts. Charts are packages of pre-configured Kubernetes resources.](https://github.com/kubernetes/helm#kubernetes-helm)

> [A chart is organized as a collection of files inside of a directory.](https://github.com/kubernetes/helm/blob/master/docs/charts.md#the-chart-file-structure)

In other words, Helm allows us to work from the mental model of managing our "application" on our cluster, instead of individual Kubernetes resources via `kubectl`.

One of the other features of Helm is its ability to use _templates_ in Kubernetes resources that are part of the chart. This means you can define values in one place and share them across multiple Kubernetes resource files.

Another good thing about a common way to deploy a set of resources in kubernetes, is that the default values will be set to whats considered best practice. This default values can of course be overridden be the ones installing the helm-chart.

Before we begin you should understand three key concepts in helm. Read them from the [docs](https://helm.sh/docs/using_helm/#three-big-concepts) or the copy-pasted text below.

### Three key concepts

A **Chart** is a Helm package. It contains all of the resource definitions necessary to run an application, tool, or service inside of a Kubernetes cluster. Think of it like the Kubernetes equivalent of a Homebrew formula, an Apt dpkg, or a Yum RPM file.

A **Repository** is the place where **charts** can be collected and shared. It’s like Perl’s CPAN archive or the Fedora Package Database, but for Kubernetes packages.

A **Release** is an instance of a chart running in a Kubernetes cluster. One chart can often be installed many times into the same cluster. And each time it is installed, a new **release** is created. Consider a MySQL chart. If you want two databases running in your cluster, you can install that chart twice. Each one will have its own **release**, which will in turn have its own **release** name.

With these concepts in mind, we can now explain Helm like this:

Helm installs **charts** into Kubernetes, creating a new **release** for each installation. And to find new **charts**, you can search Helm chart repositories.

If you want to read more about helm, I recommend the [docs](https://helm.sh/docs/).

## Installing Helm

1. Install Helm client by following the [installing-helm-guide](https://helm.sh/docs/using_helm/#installing-helm).
2. Test that your installation in successful.

```bash
$ helm version
version.BuildInfo{Version:"v3.0.3", GitCommit:"ac925eb7279f4a6955df663a0128044a8a6b7593", GitTreeState:"clean", GoVersion:"go1.13.6"}
```

3. Install helm in your cluster by running: `helm init` (assuming RBAC is not enabled in your cluster)

```bash
$ helm init
$HELM_HOME has been configured at /home/shg/.helm.
...
Happy Helming!
```

## Let's install something!

We are going to install nginx-ingress by using the [helm chart](https://github.com/helm/charts/tree/master/stable/nginx-ingress).

1. Create a folder named nginx-ingress.
2. Inside that folder create a file named values.yaml. This is the file we use to override the default settings. The default settings for this chart you can find [here](https://github.com/helm/charts/tree/master/stable/nginx-ingress).

```yaml
#values.yaml

# The only value we want to override is that we don't want to create rbac
rbac:
  create: false
```

3. Install nginx-ingress by:

```bash
$ helm upgrade --install nginx stable/nginx-ingress --values values.yaml
...
A list of all deployed resources are listed, with some useful notes...
Happy helming!
```

This will install the chart **stable/nginx-ingress**, create a release named **nginx** and override the default values with values.yaml.

### Lets create our own hostname for the nginx-ingress

1. Go to https://portal.azure.com/.
2. Find the resource group for the cluster. (Not the one you specified, but the one automatically created my azure.)
3. In this resource you will find some resources of type **Public ip address**. Go through then until you find the one for nginx-ingress. It is normally tagged with the name of the kubernetes service **nginx-nginx-ingress-controller**.
4. In the left meny click **Configuration**, then type inn your **Dns name label**, and hit save.

![KubeMQ Dashboard](images/azure-portal-dns.png)

5. Now you can reach your nginx-ingress my http://yourname.westeurope.cloudapp.azure.com. (For now it wil give you a 404).

### Useful helm commands

All helm commands is described [here](https://helm.sh/docs/helm/#helm), but some useful ones are listed below.

- [helm install](https://helm.sh/docs/helm/#helm-install) - install a helm chart.
- [helm delete](https://helm.sh/docs/helm/#helm-delete) - delete a helm release.
- [helm ls](https://helm.sh/docs/helm/#helm-status) - lists all helm releases
- [helm status](https://helm.sh/docs/helm/#helm-status) - displays the status of the named release
- [helm upgrade](https://helm.sh/docs/helm/#helm-upgrade) - upgrade a helm release

## Ingress and ingress-controllers

Nginx-ingress in an [ingress-controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/). An ingress-controller listens to changes in an kubernetes [Ingress] and apply them to the underlying proxy, in this case nginx. An [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) is a kubernetes resource that manages external access to a service inside the cluster. You can look at a Ingress as a routing rule, like so: "If the user navigates to hostname.com/tardis then redirect the user the the service named tardis-service`. Lets take our webpage as an example.

## Want some more?

If you want to create an ingress, do the steps below, or jump to next section. We are going to use the nginx-ingress to host kibana in [section 6 - ELK](6-helm-and-elk).

### Create an ingress to kubemq-dashboard

In a production environment we should place all our applications behind a proxy (nginx-ingress), but for now lets just take kubemq-dashboard as an example.

1. First we need to change the a service kubemq-dashboard the be of type NodePort. This enables clients inside the cluster to reach the kubemq-dashboard.
2. Go to ./speedtest-scheduler/Deployment/KubeMQ.yaml and change:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: kubemq-dashboard
spec:
  type: NodePort # <-- Change here!
  selector:
    app: kubemq-dashboard
  ports:
    - protocol: TCP
      port: 80
```

3. Run `kubectl apply -f ./speedtest-scheduler/Deployment/KubeMQ.yaml`
4. Then we create an Ingress that points to the service. Remember to update host.

```yaml
#ingress.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubemq-dashboard
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    kubernetes.io/ingress.class:
      nginx # This tells kubernetes that we want to use our nginx-ingress

      # Needed because of the web page don't have a basepath config.
    nginx.ingress.kubernetes.io/configuration-snippet: |
      sub_filter '<base href="/">' '<base href="/kubemq/">';
spec:
  rules:
    - host: tardis.westeurope.cloudapp.azure.com # CHANGE HERE!
      http:
        paths:
          - path: /kubemq/?(.*) # The path we want to use
            backend:
              serviceName: kubemq-dashboard # The service we want tha path to be redirected to
              servicePort: 80
```

5. Run `kubectl apply -f ./ingress.yaml`
6. Go to https://yourhostname.westeurope.cloudapp.azure.com/kubemq

## What now?

Helm makes installing third-party software on Kubernetes easier, and in the next section we'll use Helm to [install the ELK-stack](6-helm-and-elk) and monitor our system.
