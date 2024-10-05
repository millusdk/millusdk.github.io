---
layout: post
title:  "Creating a Gitlab agent"
date:   2024-10-05 06:45:00 +0200
categories: gitlab proxmox homelab docker
---
Having installed Gitlab in a previous [post]({% post_url 2024-09-27-setting-up-gitlab %}), it is now time to setup the first part of the CI/CD workflow, namely building the application.

As stated in my reasoning for choosing Gitlab in the first place, I want to keep as much as possible in the same "ecosystem". Because of this I will be using the built in CI functionality of Gitlab, built on their Gitlab runners.

There are several options for how Gitlab runners are executed. The ones that might be relevant in my setup are:
1. Shell
2. Docker
3. Docker Autoscaler
4. Kubernetes
5. Instance

Since I would like to run the CI flows as isolated as possible, I have want to setup the runner with a Docker executor.

## Setting up the LXC container
Similarly to how I am running Gitlab itself in a LXC container, I will be running the agent in its own container based on Ubuntu 22.04.

### Defining the container
Gitlab do not have any recommended specs for a runner, but I have chosen the following for my usecase:

| CPU cores | Memory | Storage |
|-----------|--------|---------|
| 2 vCores  | 4 GB   | 32 GB   |

## Installing prerequisites
Using the Docker Executor, requires that Docker is installed and that the version is greater than ```1.13.0```.

Lets install Docker on our agent. First we need to install some prerequisites and download the docker pgp key:
{% highlight  bash %}
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
{% endhighlight %}

Now we can add Dock'ers repository to apt:
{% highlight bash %}
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
{% endhighlight %}

Now lets update the list of sources for apt and install docker-ce:

{% highlight bash %}
sudo apt install apt-transport-https ca-certificates curl software-properties-common
sudo apt install docker-ce
{% endhighlight %}

We can now verify that docker is running:
{% highlight bash %}
sudo systemctl status docker
{% endhighlight %}

## Installing the runner
The installer for Gitlab runners support installation through apt, but first the repository needs to be added to apt:

{% highlight bash %}
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
{% endhighlight %}

> **CAUTION:**  Executing scripts directly from the internet is generally not recommended! You should always download and check what the script does before executing it.

Once the script has run, we are ready to install the runner:

{% highlight bash %}
sudo apt install gitlab-runner
{% endhighlight %}

## Register runner
Now we are ready to create and register our new runner. Go to the admin panel for Gitlab, and select the "Runners"  menu option on the left. On the runners page, click the button to add a new runner.

This brings up the New instance runner page. I have chosen to tag my runner as 'docker-based' to signify that it is running builds as docker containers:
![Image showing the page for creating a new Gitlab runner](/images/gitlab-agent/new-runner.png)

Once the information has been provided, click the "Create runner" button to create the new runner in Gitlab.

This takes us to the page to register our runner. Copy the provided script and run it on the agent. You will be asked for a default docker image, I have chosen to tell it to use "docker:latest".

Once this is done, Gitlab should automatically detect your new runner:
![Image showing the success message after a runner has been registered and started running in the container](/images/gitlab-agent/runner-created.png)

Clicking the "View runners" button, takes os back to the overview page for runners, but now our runner is registered and (hopefully) listed as online.

## Testing the runner
In order to test the newly created runner, I have created a new project in Gitlab for the application code. Since my prefered stack for developing applications is ASP.NET Core, I have created a simple "starter" webapp on my local computer using the built in CLI as follows:
{% highlight bash %}
dotnet new -o gitlab-test-project webapp
{% endhighlight %}

Once the project has been created, we can setup GIT for the new directory:
{% highlight bash %}
cd gitlab-test-project
dotnet new gitignore
git init --initial-branch=main
git add .
git commit -m "Initial commit of test project"
{% endhighlight %}

Now that we have the base for our project, we can add the Gitlab repository as a remote, replacing ``<url>`` with the URL from our new Gitlab repository:

{% highlight bash %}
git remote add origin <url>
git push --set-upstream origin main
{% endhighlight %}

### Creating the CI-pipeline
With the base project setup, we are now ready to create the CI-pipeline for the project. By default Gitlab uses a ``.gitlab-ci.yml`` file to define the pipeline including all steps. A simple pipeline to build our new test project can look as follows:
{% highlight yml %}
build:
  stage: build
  image: "mcr.microsoft.com/dotnet/sdk:8.0"
  only:
    - master
  script:
    - dotnet restore
    - dotnet build --configuration Release
    - dotnet publish --configuration Release --output ./publish/
  artifacts:
    paths:
      - ./publish/*
    expire_in: 1 week
{% endhighlight %}

Creating and pushing this file to the remote repository automatically triggers a build. This may take a while to complete, as it needs to pull the image we told the pipeline to use.

Once the image is downloaded, the pipeline should complete if everything has been setup correctly.

![Image showing that a Gitlab CI run has passed](/images/gitlab-agent/run-passed.png)

## Next steps
Since one of the goals for me is to be able to play around with Kubernetes, I need to be able to create docker images and store them somewhere during the CI process. To do this I need to setup Docker-in-Docker for the CI pipeline, but that is left for a post on another day.