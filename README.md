## Overview

This is a guide for using a kubernetes kind cluster in GitLab CI for testing kubernetes deployments. Kind runs a Kubernetes node that runs in a docker container.
It covers how to use Docker in Docker to host Kind and connecting to Kind from a GitLab CI job.

We start with a small example for using docker in docker service, and iterate on it such that it can be tested locally with gitlab-runner.
The GitLab CI docs describe using Docker in Docker (DinD) and other strategies for accessing docker daemon from CI for building images, however it does not demonstrate how to reach containers running inside DinD for other use cases. (e.g. Connecting to a Kind cluster running in DinD )

### Accessing Docker daemon to host Kind in gitlab-runner:
Kind runs a Kubernetes node inside a docker container. A docker daemon is required to host this container. [gitlab.com has great docs on accessing Docker daemon.](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-in-docker)
  
There are a few methods described there, we will be using the Docker in Docker service as this will work on gitlab's shared runners. Using the host's docker daemon by mounting docker socket is a viable strategy if using docker executor that mounts this socket, however gitlab's shared runners (the free shared CI) [do not mount the host's docker daemon socket.](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/17769)

It is strongly recommended to look through that doc in detail.

## Docker in Docker Service

If you look at the example configuration linked above, a docker in docker GitLab CI service looks something like this. (If you are trying to copy and paste a fully working snippet, this isn't it, scroll down)
```yaml
variables:
  DOCKER_TLS_CERTDIR: "/certs"

services:
  - docker:20.10.16-dind

# Check if we are connected to DinD service.
test-docker-connection:
  image: docker:20.10.16
  script:
    - docker info
```

### Testing this locally

Iterating on CI is extremely painful if you cannot iterate on it locally, so let's start with that.
You should install [gitlab-runner](https://docs.gitlab.com/runner/install/) to test locally. We will use the docker executor. I am assuming you have docker running already, if not, set that up.

> **Warning:**
> gitlab-runner can be a bit funky, if I describe issues with it, they may or may not be universal. 
> In case they are platform specific, I am using gitlab-runner version 15.11.0 (darwin/arm64).

Make sure that GitLabCI yaml is in .gitlab-ci.yml, if you are doing this in a project with multiple GitLab CI yaml files included in .gitlab-ci.yml, note that [gitlab-runner exec will not include these files.](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/27205)

gitlab-runner requires a git repository. gitlab-runner will likely give you a warning about needing to check in changes in your git repo, if your CI does not use anything outisde of .gitlab-ci.yml (like in these examples), maybe you can ignore this, but if things seem broken, try committing the change. Ok that's it for gitlab-runner exec warnings, good luck...


The name of our job here is `test-docker-connection`.

Run the job on docker executor locally:

```
gitlab-runner exec docker test-docker-connection
```

This will fail as gitlab-runner's default configuration does not match what was specified in docs on gitlab.com. You should expect logs showing the DinD service initialize, generate certs for tls, and then some errors:

```
...
2023-05-30T22:56:08.667315804Z Getting CA Private Key
2023-05-30T22:56:08.675997137Z /certs/client/cert.pem: OK
2023-05-30T22:56:08.810817012Z ip: can't find device 'ip_tables'
2023-05-30T22:56:08.812551470Z modprobe: can't change directory to '/lib/modules': No such file or directory
2023-05-30T22:56:08.816443179Z mount: permission denied (are you root?)
2023-05-30T22:56:08.816549929Z Could not mount /sys/kernel/security.
2023-05-30T22:56:08.816589054Z AppArmor detection and --privileged mode might break.
2023-05-30T22:56:08.818090054Z mount: permission denied (are you root?)
...
```

Docker DinD container requires privileges it does not currently have. If you read the gitlab docs, you may have noticed they include --docker-privileged, this is required.

```
gitlab-runner exec docker test-docker-connection --docker-privileged
```

> **Warning:**
> We are delibarately configuring gitlab-runner with cli and not its config.toml for local testing.
> After much pain I found out gitlab-runner exec does not read this config.
> (https://gitlab.com/gitlab-org/gitlab/-/issues/216301)


Our DinD service initializes, and the job now runs and provides this output:

```
ERROR: Cannot connect to the Docker daemon at tcp://docker:2375. Is the docker daemon running?
```

By default, the docker image used in our job is trying to connect to docker daemon with hostname 'docker'. Our service's hostname is also 'docker', as it is generated from the image name by GitLab CI ([unless you aliased it](https://docs.gitlab.com/ee/ci/services/#define-services-in-the-gitlab-ciyml-file)).

The hostname is correct, however port 2375 is for an unsecure connection. The DinD service is listening with TLS (on port 2376). If you set DOCKER_TLS_CERTDIR to an empty string, you should be able to connect over 2375. We are going to tell the client to connect to 2376 with the DOCKER_HOST variable. Gitlab's docs indicate doing otherwise may get you in trouble with kubernetes executor /shrug.

Below we add DOCKER_HOST and a couple other settings required by client. These pop up in the gitlab doc, under [kubernetes executor for DinD](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#docker-in-docker-with-tls-enabled-in-kubernetes) 

```yaml
variables:
  DOCKER_TLS_CERTDIR: "/certs"
  # Connect to https port on docker daemon
  DOCKER_HOST: docker:2376
  # Docker client uses https instead of http
  DOCKER_TLS_VERIFY: "1"
  # Mount /certs/client, docker client needs these certs to connect.
  DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"

```

Run it again, new error:

```
unable to resolve docker endpoint: open /certs/client/ca.pem: no such file or directory
```

The DinD service is generating certs at /certs, however this volume is not mounted, the client cannot access the certs. We add a /certs volume, which will be bound to both our job and service containers. 

```
gitlab-runner exec docker test-docker-connection --docker-privileged  --docker-volumes /certs
```

If you're lucky and nothing has gone wrong, you should be getting output from `docker info` indicating a successful connection to the docker daemon. The full .gitlab-ci.yml snippet below, and gitlab-runner commmand above.

```yaml
variables:
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_HOST: docker:2376
  DOCKER_TLS_VERIFY: "1"
  DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"

services:
  - docker:20.10.16-dind

test-docker-connection:
  image: docker:20.10.16
  script:
    - docker info
```

## Using Host's Docker Daemon (Alternative Strategy)

This method mounts the hosts's docker daemon socket instead of using docker in docker. This [will not work](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/17769) on gitlab's shared runners. The host must have a docker daemon running. (i.e. method is restricted to a gitlab-runner using Docker executor).The running GitLab CI job will be able to view and modify all of the hosts Docker resources. DinD service should probably be the first choice, but this is quick and easy if you wish to go down this path.

### Mounting Docker socket

The [Docker executor must mount](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-the-docker-executor-with-docker-socket-binding) `/var/run/docker.sock` so our job may communicate with host's daemon.

Included this in our exec command to expose daemon socket. The following snippets should successully connect to host's docker daemon.

```
gitlab-runner exec docker test-docker-connection --docker-volumes /var/run/docker.sock:/var/run/docker.sock
```

```yaml
test-docker-connection:
  image: docker:20.10.16
  script:
    - docker info

```


## Running Kind + Using in Practice
We can connect to a docker daemon from inside GitLab CI job, now going to deploy Kind. Our job should install Kubectl and Kind before running. You may wish to update the versions installed. (I am omitting DinD service / variables in the following snippets.)

```yaml
kind-test:
  image: docker:20.10.16
  before_script:
    # Install kubectl
    - wget "https://dl.k8s.io/release/v1.27.1/bin/linux/amd64/kubectl"
    - chmod +x kubectl
    - mv kubectl /usr/local/bin
    # Install Kind
    - wget "https://kind.sigs.k8s.io/dl/v0.18.0/kind-linux-amd64"
    - mv ./kind-linux-amd64 /usr/local/bin/kind
    - chmod +x /usr/local/bin/kind
  script:
    - kind --version
    - kubectl version
```

### Creating Kind Cluster

Create a kind cluster with default configuration named `test-cluster`. After the job, we delete the kind cluster, otherwise it will linger. This may be less of an issue if using DinD service, however if using host's daemon, if not deleted kind will encounter a name conflict on subsequent runs. You should see the cluster's control plane running in docker if using host's daemon.

```yaml
kind-test:
  image: docker:20.10.16
  before_script:
    # Install kubectl
    - wget "https://dl.k8s.io/release/v1.27.1/bin/linux/amd64/kubectl"
    - chmod +x kubectl
    - mv kubectl /usr/local/bin
    # Install Kind
    - wget "https://kind.sigs.k8s.io/dl/v0.18.0/kind-linux-amd64"
    - mv ./kind-linux-amd64 /usr/local/bin/kind
    - chmod +x /usr/local/bin/kind
  script:
    - kind create cluster --name test-cluster --wait 60s
    - cat "$HOME/.kube/config"
    - kubectl get pod
  after_script:
    # Ensure our kind container is cleaned up after job.
    - kind delete cluster --name test-cluster
```


### Connecting to Kind Cluster

We print the kind cluster's generated kube config, and try to get pods with kubectl.
The error below inidcates we cannot connect to the cluster with Kind's generated config. 

```
The connection to the server 127.0.0.1:40323 was refused - did you specify the right host or port?
 
```

We are attempting to reach Kind on localhost. We need to connect to the DinD service such that we can reach control plane container running inside of it. Let's take a closer look by changing our script to the following:

```yaml
  script:
    - kind create cluster --name test-cluster --wait 60s
    - cat "$HOME/.kube/config"
    - docker container ls
```

Output should look something like this:

```
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                       NAMES
db0d14e647fd   kindest/node:v1.26.3   "/usr/local/bin/entr…"   28 seconds ago   Up 27 seconds   127.0.0.1:41665->6443/tcp   test-cluster-control-plane
```

We can see that the container exposes port 6443 (Kubernetes API server) to 41665, which matches the port in our kubeconfig generated by Kind. There's a problem here, we can see that the port 41665 is only exposed to 127.0.0.1. This is NOT localhost of the gitlab job, this is localhost inside the DinD service, which explains why our kubeconfig fails to connect. We need this port exposed to our job, we will update the kind cluster's config to expose this port to any incoming address. Updated script: 

```yaml
script:
  # Configure Kind cluster
  - |
    echo -e "
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    networking:
      # Allow connection to api server outside of DinD container.
      apiServerAddress: 0.0.0.0
    " > config.yaml
  - kind create cluster --name test-cluster --wait 60s --config config.yaml
  - cat "$HOME/.kube/config"
  - docker container ls
  - kubectl get pods
```

Make sure to verify the kind container's port is now exposed to 0.0.0.0:

```
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                     NAMES
9067abea5e8d   kindest/node:v1.26.3   "/usr/local/bin/entr…"   29 seconds ago   Up 27 seconds   0.0.0.0:38437->6443/tcp   test-cluster-control-plane
```

We have a new error from kubectl get pods: 

```
The connection to the server 0.0.0.0:38437 was refused - did you specify the right host or port?
```

Our kubeconfig is now using api server as 0.0.0.0, this is not valid, we must update kube config to point to the DinD service. If you recall from earlier, the hostname of our DinD service is generated from the image name, it is 'docker'. Let's use sed to substitute 0.0.0.0 for 'docker' hostname.

```yaml
script:
  # Configure Kind cluster
  - |
    echo -e "
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    networking:
      # Allow connection to api server outside of DinD container.
      apiServerAddress: 0.0.0.0
    " > config.yaml
  - kind create cluster --name test-cluster --wait 60s --config config.yaml
  # Update kubeconfig to connect to DinD service.
  - sed -i -E -e s/0\.0\.0\.0/docker/g "$HOME/.kube/config"
  - cat "$HOME/.kube/config"
  - kubectl get pods
```

Hopefully you are now connected to the Api Server (with a new error!). If not, double check what the server is set to after substitution in kube config, and how the port is exposed in output of docker container ls.

```
control-plane, kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster.local, localhost, not docker
Unable to connect to the server: tls: failed to verify certificate: x509: certificate is valid for test-cluster-control-plane, kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster.local, localhost, not docker
```

We are connecting to the API Server using TLS, the 'docker' hostname we are using is not included in the cluster's certificate. One option to fix this is to add the hostname to the certificate by patching the kubeadm config via kind's config. If you want to do this, [look here](https://kind.sigs.k8s.io/docs/user/configuration/#kubeadm-config-patches) for how to patch kubeadm config from kind, look at the kubeadm InitConfiguration spec, and see certSANs, which must include our hostname for secure connection.

I found this painful, I would recommend this ~~hack~~ shortcut instead. A number of hostnames were listed in that error, we are going to [alias our DinD service](https://docs.gitlab.com/ee/ci/services/#define-services-in-the-gitlab-ciyml-file) to `kubernetes`, a valid hostname. If you have a bunch of jobs sharing this DinD service, and they are using it for stuff other then kubernetes, this may be confusing, but you can figure that out.

Modify the DinD service to the following:

```yaml
services:
  - name: docker:20.10.16-dind
    alias: kubernetes
```

The hostname we are using in kubeconfig is now wrong, this should be updated to use `kubernetes`.

```
  - sed -i -E -e s/0\.0\.0\.0/kubernetes/g "$HOME/.kube/config"
```

If all went well our kubectl command should now connect to Kind without issue. You should now be able to use kubectl, helm, or whatever tools to use kubernetes in GitLab CI.

```
No resources found in default namespace.
```


## Complete Kind/DinD GitLab CI Example:

```yaml
variables:
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_HOST: docker:2376
  DOCKER_TLS_VERIFY: "1"
  DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"

services:
  - name: docker:20.10.16-dind
    alias: kubernetes

kind-test:
  image: docker:20.10.16
  before_script:
    - wget "https://dl.k8s.io/release/v1.27.1/bin/linux/amd64/kubectl"
    - chmod +x kubectl
    - mv kubectl /usr/local/bin
    - wget "https://kind.sigs.k8s.io/dl/v0.18.0/kind-linux-amd64"
    - mv ./kind-linux-amd64 /usr/local/bin/kind
    - chmod +x /usr/local/bin/kind
  script:
    - |
      echo -e "
      kind: Cluster
      apiVersion: kind.x-k8s.io/v1alpha4
      networking:
        apiServerAddress: 0.0.0.0
      " > config.yaml
    - kind create cluster --name test-cluster --wait 60s --config config.yaml
    - sed -i -E -e s/0\.0\.0\.0/kubernetes/g "$HOME/.kube/config"
    - kubectl get pods
  after_script:
    - kind delete cluster --name test-cluster
```


And to run locally: 

```
gitlab-runner exec docker kind-test --docker-privileged  --docker-volumes /certs
```

## Conclusion

Hopefully this was useful and you can figure out whatever crap pops up that isn't covered here or in GitLab's docs. I'm sure pieces of this will break in newer versions, but it's at least somewhere to start. Have fun...



