# Tekton in Action

In this guide you can learn how to install Tekton and how to create a simple Task and Pipeline. 

[Tekton](https://tekton.dev) is a non-opinionated Pipeline Engine built for the cloud (more specifically, for Kubernetes). You can build any kind of pipeline that you want as the engine doesn't impose any restrictions on the kind of Tasks that it can execute. This makes it perfect for building Service Pipelines where you might have special requirements that cannot be met by a managed service.  

After running our first Tekton Pipeline, you'll find in this tutorial links to more complex Service Pipelines used to build the Conference Application Services.

## Installing Tekton

Follow the next steps in order to install and setup Tekton in your Kubernetes Cluster. 

> [!NOTE]
> If you don't have a Kubernetes Cluster you can create one with [KinD, as we did on Chapter 2](../../chapter-2/README.md#creating-a-local-cluster-with-kubernetes-kind) 

1. Install Tekton Pipelines
    
    ```shell
      kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.45.0/release.yaml
    ```

### Tekton Dashboard _(optional)_

1. Install Tekton Dashboard 

    ```shell
    kubectl apply -f https://github.com/tektoncd/dashboard/releases/download/v0.33.0/release.yaml
    ```
2. To access the dashboard, you need to first do port-forwarding with `kubectl`:

    ```shell
    kubectl port-forward svc/tekton-dashboard  -n tekton-pipelines 9097:9097
    ```
1. Then, you should be able to access the dashboard in your browser using the url: [http://localhost:9097](http://localhost:9097)

    ![Tekton Dashboard](imgs/tekton-dashboard.png)

### Tekton CLI _(optional)_

You can also install [Tekton `tkn` CLI tool](https://github.com/tektoncd/cli). 
If you are using Mac OSX, you can run: 

```shell
brew install tektoncd-cli
```

## Getting started with Tekton Tasks

This section aims to get you started creating Tasks and a Simple Pipeline. Then, you should be able to look into the Service Pipelines used to build the artifacts for the Conference Application. 

With Tekton we can define what each task does by creating a Tekton Task definitions. 

### Creating a task

1. In the [`hello-world/hello-world-task.yaml`](hello-world/hello-world-task.yaml), we have the simplest example of a task definition.
    
    ```yaml
    apiVersion: tekton.dev/v1
    kind: Task
    metadata:
      name: hello-world-task
    spec:
      params:
      - name: name
        type: string
        description: who do you want to welcome?
        default: tekton user
      steps:
        - name: echo
          image: ubuntu
          command:
            - echo
          args:
            - "Hello World: $(params.name)" 
    ```

    This Tekton `Task` uses the `ubuntu` image and the `echo` command located inside that image. This `Task` also accept a parameter called `name` that will be used to print a message. 

3. Let's apply this `Task` definition to our cluster by running: 

    ```shell
    kubectl apply -f hello-world/hello-world-task.yaml
    ```

    > [!NOTE] 
    > When we apply this resource to Kubernetes we are not executing the task, we are only making the Task definition available for others to use it. 

    This task can now be referenced in multiple pipelines or be executed independently, by different users. 

1. List the available tasks in the cluster by running: 
    ```shell
    > kubectl get tasks
    NAME               AGE
    hello-world-task   88s
    ```

### Using TaskRun to execute a Task 

1. Next, to run our Task, we need to create a `TaskRun` resource. Each  `TaskRun` represents a single run for a task. This is what we have in the [hello-world/task-run.yaml](hello-world/task-run.yaml):

    ```yaml
    apiVersion: tekton.dev/v1
    kind: TaskRun
    metadata:
      name: hello-world-task-run-1
    spec:
      params: 
      - name: name
        value: "Building Platforms on top of Kubernetes reader!"
      taskRef:
        name: hello-world-task
    ```
   Notice that this concrete run has a fixed resource name (`hello-world-task-run-1`) and a concrete value for the task parameter called `name`.

1. To create our first Task Run (execution), apply this [`TaskRun`](hello-world/task-run.yaml) resource to the cluster:
    ```shell
    kubectl apply -f hello-world/task-run.yaml
    taskrun.tekton.dev/hello-world-task-run-1 created
    ```

    As soon as the `TaskRun` is created, the Tekton Pipeline Engine is in charge of scheduling the tasks and creating the Kubernetes Pod needed to execute it. 
2. If you list the pods in the default namespace you should see something like this: 

    ```shell
    kubectl get pods
    NAME                         READY   STATUS     RESTARTS   AGE
    hello-world-task-run-1-pod   0/1     Init:0/1   0          2s
    ```

3. Also, to check the status of a `TaskRun`, you can list them: 

    ```shell
    kubectl get taskrun
    NAME                     SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
    hello-world-task-run-1   True        Succeeded   66s         7s
    ```

4. Finally, because we were executing a single task you can see the logs of the `TaskRun` execution by tailing the logs of the pod that was created:
    
    ```shell
    kubectl logs -f hello-world-task-run-1-pod 
    Defaulted container "step-echo" out of: step-echo, prepare (init)
    Hello World: Building Platforms on top of Kubernetes reader!
    ```

Awesome, you've created and executed a task within Kubernetes using Tekton! Let's now look into how to sequence multiple tasks together using a Tekton Pipeline.

## Getting Started with Tekton Pipelines

Let's use a Pipeline to coordinate multiple tasks, like the one that we defined before. There are reusable `Task` definitions, created by the Tekton community, available in the [Tekton Hub](https://hub.tekton.dev/). 

![Tekton Hub](imgs/tekton-hub.png)

### Installing an existing Task from the Tekton Hub
1. Before creating the Pipeline we need to install the `wget` Tekton task, from the Tekton Hub, by running: 

    ```shell
    kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/wget/0.1/wget.yaml
    ```

1. You should see: 
    
    ```shell
    task.tekton.dev/wget created
    ```

Now, let's use our `Hello World` task and the `wget` task, that we have just installed, together into a simple pipeline. 

### Using Pipelines in Tekton

The Pipeline Definition we'll create, should be able to fetch a file, read its content and then use the `Hello World` Task that was previously defined .

![Hello World Pipeline](imgs/hello-world-pipeline.png)

#### Creating a Pipeline

1. Let's create the following Pipeline Definition, [hello-world-pipeline ](hello-world/hello-world-pipeline.yaml):

    ```yaml
    apiVersion: tekton.dev/v1
    kind: Pipeline
    metadata:
      name: hello-world-pipeline
      annotations:
        description: |
          Fetch resource from internet, cat content and then say hello
    spec:
      results: 
      - name: message
        type: string
        value: $(tasks.cat.results.messageFromFile)
      params:
      - name: url
        description: resource that we want to fetch
        type: string
        default: ""
      workspaces:
      - name: files
      tasks:
      - name: wget
        taskRef:
          name: wget
        params:
        - name: url
          value: "$(params.url)"
        - name: diroptions
          value:
            - "-P"  
        workspaces:
        - name: wget-workspace
          workspace: files
      - name: cat
        runAfter: [wget]
        workspaces:
        - name: wget-workspace
          workspace: files
        taskSpec: 
          workspaces:
          - name: wget-workspace
          results: 
            - name: messageFromFile
              description: the message obtained from the file
          steps:
          - name: cat
            image: bash:latest
            script: |
              #!/usr/bin/env bash
              cat $(workspaces.wget-workspace.path)/welcome.md | tee /tekton/results/messageFromFile
      - name: hello-world
        runAfter: [cat]
        taskRef:
          name: hello-world-task
        params:
          - name: name
            value: "$(tasks.cat.results.messageFromFile)"
    ```

As you can see, at the end it's not that easy to fetch a file, read its content and then use our hello-world task to print this fetched file's content. 

With pipelines, we have the flexibility to connect tasks, that can leverage the previous task's output in its processing. If we have more requisites in this scenario, for instance, we could add new tasks to do transformations or further processing of the inputs and outputs of each individual tasks. 

This example relies on the `wget` Task that we installed from the Tekton Hub and a task that is defined inline called `cat`. Since the `cat` task fetches the content of the downloaded file and stores it into a Tekton Result, we can then reference it into our `hello-world-task`. 

1. Now, let's install the pipeline definition: 

```shell
kubectl apply -f hello-world/hello-world-pipeline.yaml
```

#### Running a Pipeline

1. For every execution of this pipeline, we must create a new `PipelineRun`. Here's the [pipeline-run](hello-world/pipeline-run.yaml) definition we can use to run our pipeline:

    ```yaml
    apiVersion: tekton.dev/v1
    kind: PipelineRun
    metadata:
      name: hello-world-pipeline-run-1
    spec:
      workspaces:
        - name: files
          volumeClaimTemplate: 
            spec:
              accessModes:
              - ReadWriteOnce
              resources:
                requests:
                  storage: 1M 
      params:
      - name: url
        value: "https://raw.githubusercontent.com/salaboy/salaboy/main/welcome.md"
      pipelineRef:
        name: hello-world-pipeline
      
    ```

There are tasks in this pipeline download and store files in the filesystem, therefore we are using Tekton workspaces as an abstraction to provide storage for our `PipelineRun`s.

As we did before with our `TaskRun`, we can also provide parameters for the `PipelineRun`. This customization allows us to parameterize each run to with different configurations, or in this case different files.

> [!NOTE]
> For both `PipelineRuns` and `TaskRuns`, each run will require a new resource name. Therefore, even if you try to apply the same resource twice, the Kubernetes API server will not allow you to change the existing resource, since it has the same name.

1. Now, run the pipeline by executing: 
    
    ```shell
    kubectl apply -f hello-world/pipeline-run.yaml
    ```

1. Check the pods that are created: 
    
    ```shell
    > kubectl get pods
    NAME                                         READY   STATUS        RESTARTS   AGE
    affinity-assistant-ca1de9eb35-0              1/1     Terminating   0          19s
    hello-world-pipeline-run-1-cat-pod           0/1     Completed     0          11s
    hello-world-pipeline-run-1-hello-world-pod   0/1     Completed     0          5s
    hello-world-pipeline-run-1-wget-pod          0/1     Completed     0          19s
    ```

    Notice that there is one Pod per Task, and a pod called `affinity-assistant-ca1de9eb35-0` which is making sure that the Pods are created in the correct node (where the volume was bound).

1. Check the TaskRun too: 
    
    ```shell
    > kubectl get taskrun
    NAME                                     SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
    hello-world-pipeline-run-1-cat           True        Succeeded   109s        104s
    hello-world-pipeline-run-1-hello-world   True        Succeeded   103s        98s
    hello-world-pipeline-run-1-wget          True        Succeeded   117s        109s
    ```

1. And of course, if all the tasks are successful, the PipelineRun will be too: 

    ```
    kubectl get pipelinerun
    NAME                         SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
    hello-world-pipeline-run-1   True        Succeeded   2m13s       114s
    ```

1. If you [installed the Tekton Dashboard](#tekton-dashboard-optional),  make sure to check the pipeline and task executions in the Web UI [http://localhost:9097/#/pipelineruns](http://localhost:9097/#/pipelineruns):

   ![Tekton Dashboard](imgs/tekton-dashboard-hello-world-pipeline.png)


## Tekton for Service Pipelines

Service Pipelines in real life are much more complex than the previous  examples. In the real world, it's common to have scenarios where a task requires special configurations and credentials to accessing external systems. 

In this same directory, you can find an example of a Service Pipeline definition, in the file called [service-pipeline.yaml](service-pipeline.yaml).

![Service Pipeline](imgs/service-pipeline.png)

The example Service Pipelines uses [`ko`] to build and publish the container image for our Service. This pipeline is very specify to our Go Services. If we were building services developed with another programming language, we would need to use different tools. The example service pipeline can be parameterized to build different services.

## Configuring Docker Hub Credentials 

To run this Service Pipeline you need to set up credentials to a Container Registry. This is what will allow the pipelines to push container images to a container registry such as Docker Hub. 

> [!NOTE] 
> To authenticate with a container registry from a Tekton Task/Pipeline [check the official documentation](https://tekton.dev/docs/how-to-guides/kaniko-build-push/#container-registry-authentication).

1. For this example, we should create a Kubernetes Secret containing our Docker Hub credentials:
    
    ```shell
    kubectl create secret docker-registry docker-credentials --docker-server=https://index.docker.io/v1/ --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
    ```

1. Then, install the `Git Clone` and the `ko` Tekton Tasks: 
    
    ```shell
    kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
    kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/ko/0.1/ko.yaml
    ```

1. Next, install the Service Pipeline definition, [service-pipeline.yaml](service-pipeline.yaml), in the Kubernetes cluster:
    
    ```shell
    kubectl apply -f service-pipeline.yaml
    ```

1. To build and publish our services container images, create new pipeline instances. The following `PipelineRun` resource configures the Service Pipeline, so that it can build the Notifications Service.
    
    > [!IMPORTANT]
    > You need to modify the `spec.params` section to allow the pipeline to push the resulting container image to your own registry. To do so, replace `docker.io/salaboy` with your registry + username.
    > 

    ```yaml
    apiVersion: tekton.dev/v1
    kind: PipelineRun
    metadata:
      name: service-pipeline-run-1
      annotations:
        kubernetes.io/ssh-auth: kubernetes.io/dockerconfigjson
    spec:
      params:
      - name: target-registry
        value: docker.io/salaboy
      - name: target-service
        value: notifications-service
      - name: target-version 
        value: 1.0.0-from-pipeline-run
      workspaces:
        - name: sources
          volumeClaimTemplate: 
            spec:
              accessModes:
              - ReadWriteOnce
              resources:
                requests:
                  storage: 100Mi 
        - name: docker-credentials
          secret:  
            secretName: docker-credentials
      pipelineRef:
        name: service-pipeline
    ```

    Using the `target-service` parameter, you can define which service this pipeline should build (_the conference application has the available services: `notifications-service`, `agenda-service`, `c4p-service`, `frontend`_)

   1. Create a new instance of the Service Pipeline by applying the `PipelineRun` definition [service-pipeline-run.yaml](service-pipeline-run.yaml) to the cluster, as shown below: 
    
       ```shell
       kubectl apply -f service-pipeline-run.yaml
       ```

### Using Helm Charts 

There is a separate pipeline ([app-helm-chart-pipeline.yaml](app-helm-chart-pipeline.yaml)) that package and publish the Helm Chart which includes all the application services. 

When the team decides the combination of services and version should be bundled inside a helm chart, they can use another pipeline to package and publish the chart to the same container registry where the services' container images are published. 

![Helm Chart Application Pipeline](imgs/app-helm-pipeline.png)

1. To install the Application Helm Chart Pipeline, execute: 
    
    ```shell
    kubectl apply -f app-helm-chart-pipeline.yaml
    ```

1. You can now run the pipelines, in other words, create new instances, by creating new `PipelineRun` resources:
    
    ```yaml
    apiVersion: tekton.dev/v1
    kind: PipelineRun
    metadata:
      name: app-helm-chart-pipeline-run-1
      annotations:
        kubernetes.io/ssh-auth: kubernetes.io/dockerconfigjson
    spec:
      params:
      - name: target-registry
        value: docker.io/salaboy
      - name: target-version
        value: v0.9.9
      workspaces:
        - name: sources
          volumeClaimTemplate: 
            spec:
              accessModes:
              - ReadWriteOnce
              resources:
                requests:
                  storage: 100Mi 
        - name: dockerconfig
          secret:  
            secretName: docker-credentials
      pipelineRef:
        name: app-helm-chart-pipeline
    ```

1. Now, create create a new instance of the Application Helm Chart Pipeline. To do so, apply this `PipelineRun` definition to the Kubernetes cluster:

    ```shell
    kubectl apply -f app-helm-chart-pipeline-run.yaml
    ```

- The Application Helm Chart pipeline also uses the existing `docker-credentials` to push the Helm Chart as an OCI container image. 
- The pipeline accepts the `target-version` parameter, which is used to patch the `Chart.yaml` file before packaging and pushing the helm chart to the OCI container registry. 
- Notice that this pipeline doesn't patch the container's versions that are referenced by the chart. This means that is it up to you to adapta the pipeline so that it can accept, as parameters, the versions of each service, and can then validate if the configured images exist in the referenced container registry.


> [!Note]
> These pipelines are just examples to illustrate the work required to configure Tekton, to build containers and charts.
> 
> Looking closely, you would be able to observe that the Application Helm Chart Pipeline doesn't change the version of the chart or the version of the container images configured inside the chart. 
> 
> If we really want to automate all the process, we should be able to retrieve the image's and chart's versions from a Git tag that represents the version that we want to release.

## Clean up

If you want to get rid of the KinD Cluster created for these tutorials, remember to run:

```shell
kind delete clusters dev
```

## What's next? 

Now that you have tried out Tekton, what about giving a look at Dager? 

To keep going with our plan described on the [Chapter 3 guides](../README.md#chapter-3--service-pipelines), you can now get to some hands-on practices using [Dagger for Service Pipelines](../dagger/README.md)
