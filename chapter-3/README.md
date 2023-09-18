# Chapter 3 :: Service Pipelines

In this guide, you'll learn about both Tekton and Dagger for handling Service Pipelines. 

> [!Important]
> I strongly recommend following the tutorials listed for Tekton and Dagger in your local environment.

See below the links and descriptions of each guide:

- [Tekton for Service Pipelines Tutorial](tekton/README.md)

    _Extend the Kubernetes APIs to define Pipelines and Tasks._

- [Dagger for Service Pipelines Tutorial](dagger/README.md)

    _Programmatically define Pipelines that can be executed remotely in Kubernetes or locally in our development environment._

- [GitHub Actions](github-actions/README.md)

    _A set of GitHub Actions is to allow you to compare between the two previous different approaches._


Thanks to the fantastic cloud-native community, you can read these tutorials in the following languages:
- [Chinese zh-cn](README.zh-cn.md) 🇨🇳

## Clean up

If you want to get rid of the KinD Cluster created for these tutorials, you can run:

```shell
kind delete clusters dev
```

## Next Steps

- If you have a development background you can extend the Dagger pipelines with your custom steps;
- If you are not a Go developer, would you dare to build a pipeline for your tech stack using Tekton and Dagger? 
- If you have a container registry account (e.g. Docker Hub account):
  - Try configuring the credentials for the pipelines so that they can push the container images to the registry. 
  - Then, use the Helm Chart `values.yaml` to consume the images from your account, instead of the official ones hosted in `docker.io/salaboy`. 
  - Finally, you can fork this repository `salaboy/platforms-on-k8s` in your own GitHub user (organization) to try out the GitHub actions located in this directory [../../.github/workflows/](../../.github/workflows/).


## Sum up and Contribute

In these tutorials we experimented with two completely different approaches for Service Pipelines. We started with [Tekton](https://tekton.dev) a non-opinionated Pipeline Engine that was designed to be Kubernetes-native, leveraging the declarative power of the Kubernetes APIs and Kubernetes reconciliation loops. Then we used [Dagger](https://dagger.io) a pipeline engine designed to orchestrate containers and that can be configured using your favourite tech stack and using their SDKs. 

One thing is for sure, no matter which Pipeline Engine your organization has chosen - development teams will greatly benefit from being able to consume Service Pipelines without the need of defining every single step, credentials and settings' details. If the platform team can create shared steps/tasks, or even default pipelines, for your services, developers should then be able to focus on writing application code. 

Do you want to improve this tutorial? Create an [issue](https://github.com/salaboy/platforms-on-k8s/issues/new), message me on [Twitter](https://twitter.com/salaboy), or send a [Pull Request](https://github.com/salaboy/platforms-on-k8s/compare).