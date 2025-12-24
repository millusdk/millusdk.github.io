---
layout: post
title:  "Setting up authentication in Kubernetes to Harbor"
date:   2025-12-24 06:45:00 +0200
categories: proxmox homelab kubernetes harbor
---

For my homelab I have setup [Habor](https://goharbor.io/) as my container registry. Harbor is, like Flux, a CNCF graduate project.
i have made as standard Harboer setup in an LXC container.

I want to integrate my Kubernetes cluster with my Harbor container registry, such that I can deploy images from there.
One smart feature of Harbor, is that it can be configured as a pull-through cache (proxy cache in Harbor nomenclature), meaning that it can be se to fetch images from external remotes, after which they are cached. In my setup, I have configured a proxy cache called dockerhub, which points to the actual Docker Hub registry.

Once the proxy cache has been created I have created a global robot account in Harbor, meaning that the same credentials can be used for both the proxy cache, as well as any other projects I might create later. In thie step, it is imporant to note the password that Harbor creates for the robot account, as it is needed in the next step.

We are now ready to create a secret in Kubernetes with the credentials needed to authenticate against Harbor:

{% highlight bash %}
kubectl create secret docker-registry harborcreds \
 --docker-server=<harbor registry url> \
 --docker-username=<harbor robot account name> \
 --docker-password=<harbor robot password>
{% endhighlight %}

Finally, I can update the sample repository from the previous [post]({% post_url 2025-12-22-fluxcd %}) to pull the whoami image from Harbor rather than from Docker Hub:

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      imagePullSecrets:
        - name: harborcreds
      containers:
        - name: whoami
          image: <harbor url>/dockerhub/traefik/whoami
          ports:
            - containerPort: 80
{% endhighlight %}

After this, all tha tis needed is to wait for Flux to pickup the updated configuration and reconcile it with the cluster.