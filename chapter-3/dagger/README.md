# Chapter 3 :: Dagger in Action

In this short tutorial we will be looking at how to use the Dagger Service Pipelines to build, test, package and publish each service. 

These pipelines are implemented in Go using the Dagger Go SDK and take care of building each service and creating a container. A separate pipeline is provided for building and publishing the application's Helm Chart.

## Pre-requisites

To run these pipelines locally you will need: 
- [Go installed](https://go.dev/doc/install)
- [A container runtime (such as Docker, running locally)](https://docs.docker.com/get-docker/)

To run the pipelines remotely on a Kubernetes Cluster you can use [KinD](https://kind.sigs.k8s.io/) or any Kubernetes Cluster that you have available. 

## Let's run some pipelines

Since the services in this guide are very similar, we can parameterize the same pipeline definition and use it to build each service separately. 

To run the pipelines locally, you need to clone this repository and access the [Conference Application directory](../../conference-application/).  

You can also run any defined task inside the `service-pipeline.go` file:

```shell
go mod tidy
go run service-pipeline.go build <SERVICE DIRECTORY>
```

You can find the following tasks defined for all the services: 
- `build`:  
  - **What it does**: Builds the service source code and creates a container for it. 
  - **Required arguments**: The directory of the service that we want to build.
- `test`: 
  - **What it does**: Tests the service, but first it will start all the service's dependencies (containers needed to run the tests).
  - **Required arguments**: The directory of the service that we want to test.
- `publish`: 
  - **What it does**: Publishes the created container image to a container registry. This requires you to be logged in to the container registry and to provide the name of the tag used for the container image. 
  - **Required arguments**: The directory of the service that we want build and publish, and the tag used to stamp the container before pushing it.

You can safely run `go run service-pipeline.go build notifications-service` which doesn't require you to set any credentials. You can set up your container registry and username using envorionment variables, for example:

> [!important]
> If you run `go run service-pipeline.go all notifications-service v1.0.0-dagger` all the tasks will be executed. In this case, before being able to run all the tasks, you must assure that you have all the pre-requisites set, as for pushing to a Container Registry you will need to provide the appropriate credentials.

```shell
CONTAINER_REGISTRY=<YOUR_REGISTRY> CONTAINER_REGISTRY_USER=<YOUR_USER> go run service-pipeline.go publish notifications-service v1.0.0-dagger
```

Make sure to log in to the registry where you want to publish your container images to, as it's required for this to work.

Now, for development purposes, this is quite convenient, because you can now build your service code in the same way that your CI (Continuous Integration) system will do. But you don't want to run in production container images that were created in your developer's laptop right? 

The next section demonstrates a simple setup of running Dagger pipelines remotely, inside a Kubernetes Cluster. 

## Running your pipelines remotely on Kubernetes

The Dagger Pipeline Engine can be run wherever you can run containers, that means that it can run in Kubernetes without the need of complex setups. 

> [!NOTE]
> For this tutorial you need to have a Kubernetes Cluster. You can create one using [KinD as we did in Chapter 2](../../chapter-2/README.md#creating-a-local-cluster-with-kubernetes-kind).

In this tutorial, we will execute the pipelines that we ran in a local container runtime, but now, we'll execute them remotely against a Dagger Pipeline Engine that runs inside a Kubernetes Pod. 

> [!Warning]
> This is an experimental feature, and is not a recommended way to run Dagger.  

Let's run the Dagger Pipeline Engine inside Kubernetes by creating a Pod with Dagger: 

```shell
kubectl run dagger --image=registry.dagger.io/engine:v0.3.13 --privileged=true
```

Alternatively, you can apply the `chapter-3/dagger/k8s/pod.yaml` manifest using `kubectl apply -f chapter-3/dagger/k8s/pod.yaml`.

Check that the `dagger` pod is running: 
```shell
kubectl get pods 
```

As a result, you should see something like this: 
```shell
NAME     READY   STATUS    RESTARTS   AGE
dagger   1/1     Running   0          49s
```

> [!Note]: 
> This is far from ideal because we are not setting any persistence or replication mechanism for Dagger itself, therefore all the caching mechanisms are volatile in this case. Check the official documentation for more about this. 

Now to run the projects pipelines against this remote service simply export the following environment variable: 
```shell
export _EXPERIMENTAL_DAGGER_RUNNER_HOST=kube-pod://<podname>?context=<context>&namespace=<namespace>&container=<container>
```

Where:
* `<podname>` is `dagger` (because we created the pod manually), 
* `<context>` is your Kubernetes Cluster context, if you are running against a KinD Cluster this might be `kind-dev`. To find your current context name, run `kubectl config current-context`. 
* Finally `<namespace>` is the namespace where you run the Dagger Container, and 
* `<container>` is once again `dagger`. 

For my setup against KinD, the configuration would look like this: 

```shell
export _EXPERIMENTAL_DAGGER_RUNNER_HOST="kube-pod://dagger?context=kind-dev&namespace=default&container=dagger"
```

Notice also that at this point, my KinD cluster (named `kind-dev`) didn't had anything related to Pipelines. 

Now you can run the following command in any of the projects: 
```shell
go run service-pipeline.go build notifications-service
```

Or test your service remotely:
```shell
go run service-pipeline.go test notifications-service
```

In a separate terminal tab, you can tail the logs from the Dagger engine by executing the command below: 
```shell
kubectl logs -f dagger
```

With this, you can expect the build to happen remotely, inside the Cluster! 

If you were running this against a remote Kubernetes Cluster (not KinD), there will be no need for you to have a local Container Runtime to build your services and their containers. 
