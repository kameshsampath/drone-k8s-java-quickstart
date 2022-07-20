# Drone Java Quickstart

A Java quickstart to show how to use [Drone](https://drone.io) to build and deploy Java application to Kubernetes

## Pre-requisites

The following tools are required to run this quickstart,

- [Drone CLI](https://docs.drone.io/cli/install/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [kind](https://kind.sigs.k8s.io/)
- [Helm](https://helm.sh/)
- [Kustomize](https://kustomize.io/)
- [JDK 11](https://openjdk.java.net/projects/jdk/11/)
- [Maven 3.x](https://maven.apache.org/install.html)
- [yq](https://github.com/mikefarah/yq)
- [envsusbst](https://www.man7.org/linux/man-pages/man1/envsubst.1.html)

All linux distributions adds **envsubst** via [gettext](https://www.gnu.org/software/gettext/) package. On macOS it can be installed using [Homebrew](https://brew.sh/) like `brew install gettext`.

## Clone the sources

```shell
git clone https://github.com/kameshsampath/drone-java-quickstart && \
  cd "$(basename "$_" .git)"
export QUICKSTART_HOME="$PWD"
```

Going forward we will call this folder as `$QUICKSTART_HOME` in the **README**.

## Setup Environment

Follow my blog post [Yours Kindly Drone](https://kubesimplify.com/yours-kindly-drone) to prepare your local environment to run Kubernetes and Drone together.

## Add Drone Quickstart to Gitea


## Deploy Drone Docker Runner(Devel)

```shell
helm upgrade --install --devel \
  drone-runner-docker drone/drone-runner-docker \
  --namespace=drone \
  --values $QUICKSTART_HOME/helm_vars/drone-runner-docker/values.yaml \
  --wait
```

## Deploy Nexus

The Java builds will use [Sonatype Nexus](https://www.sonatype.com/products/nexus-repository) as the [Apache Maven](https://maven.apache.org/) artifacts repository manager.

```shell
kubectl apply -k "$QUICKSTART_HOME/k8s/nexus"
```

Wait for nexus to be ready,

```shell
kubectl rollout status deploy/nexus --timeout=180s
```

## Building

### Only for Kind

Update the host aliases on to have the cluster ip of `gitea-http` service,

```shell
export GITEA_HTTP_CLUSTER_IP=$(kubectl get svc gitea-http -ojsonpath='{.spec.clusterIP}')
yq -i '.host_aliases[0].ip |= env(GITEA_HTTP_CLUSTER_IP)' .drone.yml
```

### Add Steps to build application

Update the `.drone.yml` with the following additional steps that allows you to build and test the Java application,

```yaml
 - name: test-java-application
   image: docker.io/kameshsampath/drone-java-maven-plugin:v1.0.0
   pull: if-not-exists
   settings:
     maven_mirror_url: http://nexus:8081/nexus/content/groups/public/
     goals:
       - clean
       - test
     maven_modules:
       - springboot
 - name: build-java-application
   image: docker.io/kameshsampath/drone-java-maven-plugin:v1.0.0
   pull: if-not-exists
   settings:
     maven_mirror_url: http://nexus:8081/nexus/content/groups/public/
     goals:
       - clean
       - -DskipTests
       - install
     maven_modules:
       - springboot
```

### Add Step to Build Container Image

```yaml
- name: build-image
  image: gcr.io/kaniko-project/executor:debug
  commands:
    - >
      /kaniko/executor
      --context /drone/src/$MAVEN_MODULE
      --dockerfile $DOCKER_FILE 
      --destination $DESTINATION_IMAGE
  environment:
     MAVEN_MODULE: springboot
     DOCKER_FILE: Dockerfile
     DESTINATION_IMAGE: localhost:5001/example/drone-java-k8s-quickstart
```
