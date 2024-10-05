---
layout: post
title:  "Creating docker images during CI"
date:   2024-10-05 06:45:00 +0200
categories: gitlab proxmox homelab docker
---

Because I want to run my applications on Kubernetes once my lab has been setup, I need to be able to create docker containers as part of my CI flow. To do this I need to run Docker-in-Docker. Fortunately this is relatively easy with Gitlab.

In order to support Docker-in-Docker, we need to update the executor to run as privileged. To do this we need to update the configuration for the runner placed in `/etc/gitlab-runner/config.toml`.

We need to make the following change:
{% highlight toml %}
privileged = false
# change to 
privileged = true
{% endhighlight %}

After this, we need to restart the runner:
{% highlight bash %}
gitlab-runner restart
{% endhighlight %}

## Update CI to build container

If we update our yml-file as follows:
{% highlight yml %}
services:
  - name: docker:dind
    # explicitly disable tls to avoid docker startup interruption
    command: ["--tls=false"]

variables:
  DOCKER_HOST: "tcp://docker:2375"
  # Instruct Docker not to start over TLS.
  DOCKER_TLS_CERTDIR: ""
  # Improve performance with overlayfs.
  DOCKER_DRIVER: overlay2

stages:
  - .pre
  - build
  - release
  - .post

build:
  stage: build
  image: "mcr.microsoft.com/dotnet/sdk:8.0"
  only:
    - master
  script:
    - dotnet restore
    - dotnet build --configuration Release
    - dotnet publish --configuration Release --output ./publish/
    - cp Dockerfile ./publish
  artifacts:
    paths:
      - ./publish/*
    expire_in: 1 week

release:
  stage: release
  image: docker:latest
  dependencies:
    - build
  only:
    - master
  script:
    - cd publish
    - ls -la
    - DOCKER_HOST=tcp://docker:2375 docker build .
{% endhighlight %}

## Storing container images in Gitlab
We need a place to store the container images. Fortunately Gitlab has a built in support for running a container registry, it just needs to be activated.

To activate the built in registry, we need to update the configuration of Gitlab. Open `/etc/gitlab/gitlab.rb` in an editor, and uncomment/edit the following lines:

{% highlight ruby %}
registry_external_url '<registry url>'

gitlab_rails['registry_enabled'] = true
gitlab_rails['registry_host'] = "<registry url>"
gitlab_rails['registry_path'] = "/var/opt/gitlab/gitlab-rails/shared/registry"
registry['enable'] = true
registry['registry_http_addr'] = "127.0.0.1:5050"
registry['log_directory'] = "/var/log/gitlab/registry"
registry['env_directory'] = "/opt/gitlab/etc/registry/env"
{% endhighlight %}

Personally I would like the registry to use HTTPS similar to Gitlab itself, so I have added the url for my registry to the DNS of my piholes, and I have added it to the certificates that are handled with certbot on the Gitlab instance. This is done similar to how I did for Gitlab itself, as specified under the Enabling HTTPS section on [Setting up Gitlab in a homelab with HTTPS, certbot and Cloudflare]({% post_url 2024-09-27-setting-up-gitlab %}).

With the certificate created, we can update the nginx configuration for Gitlab, and uncomment/edit the following lines:

{% highlight ruby %}
registry_nginx['enable'] = true

registry_nginx['listen_port'] = 443
registry_nginx['ssl_certificate'] = "/etc/letsencrypt/live/<registry url>/fullchain.pem"
registry_nginx['ssl_certificate_key'] = "/etc/letsencrypt/live/<registry url>/privkey.pem"
{% endhighlight %}

Finally we can reconfigure Gitlab, enabling the registry:
{% highlight bash %}
gitlab-ctl reconfigure
{% endhighlight %}

## Updating the CI pipeline

We can now update the CI pipeline to generate, tag and push the image:
{% highlight yml %}
services:
  - name: docker:dind
    # explicitly disable tls to avoid docker startup interruption
    command: ["--tls=false"]

variables:
  DOCKER_HOST: "tcp://docker:2375"
  # Instruct Docker not to start over TLS.
  DOCKER_TLS_CERTDIR: ""
  # Improve performance with overlayfs.
  DOCKER_DRIVER: overlay2

stages:
  - .pre
  - build
  - release
  - .post

build:
  stage: build
  image: "mcr.microsoft.com/dotnet/sdk:8.0"
  only:
    - master
  script:
    - dotnet restore
    - dotnet build --configuration Release
    - dotnet publish --configuration Release --output ./publish/
    - cp Dockerfile ./publish
  artifacts:
    paths:
      - ./publish/*
    expire_in: 1 week

release:
  stage: release
  image: docker:latest
  dependencies:
    - build
  only:
    - master
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
    - cd publish
    - docker build -t $CI_REGISTRY_IMAGE -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE --all-tags
{% endhighlight %}

Commit and push this change to the repository, and after a short while, the build should run including both stages:
![Image showing a completed pipeline with two of two stages passed](/images/docker-in-docker/run-passed.png)

If we open the container registry for the project, we can see that the image has been pushed, and is available for deployment:

![Container registry overview in Gitlab with a single container](/images/docker-in-docker/container-registry.png)

## Next steps
Next step in my homelab journey will be to setup a Kubernetes cluster to deploy the images to.