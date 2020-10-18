# Terraform Operator Design

Below is a diagram of the basic idea of the project

![](../tfop_1.png)

The controller is responsible for fetching tfvars or other files, and then creates a Kubernetes Job to perform the actual terraform execution. By default, the Terraform-operator will save state in a Consul on the same cluster. Even though Consul is the default, other state backends can be configured.


## Terraform Operator Components

## Controller

The controller is responsible for configuring and creating the "Terraform Runner" job.

**Under the hood**

To configure the "Terraform Runner" properly, the controller must fetch files when sources are configured, consolidate tfvars, and add volumes and volumeMounts to the pod template. 

Some examples that sources are used for are to fetch templates or tfvar files. These files can only be fetched from a git repo at this time. (See [Fetching Modules and Other Files](modules-and-configs.md).) Once the files are downloaded, the controller adds them to a ConfigMap so they can be mounted as a Volume in the "Terraform Runner".

> Since a ConfigMap has a size limitation, the files fetched from `spec.sources` should be simple text-based files. Binaries should be avoided.


## Terraform Runner

This job triggers a Pod that runs once to completion. The pod downloads the terraform module, defined from the resource, into the `/main_module` directory on the pod. Once the module is downloaded, it copies all the files from the ConfigMap which was generated by the controller into `/main_module` as well. 

Because `/main_module` is where ConfigMap files get placed, a user can reason about where their template file will be in relation to the module path.

### Outputs

When the "Terraform Runner" Pod is done, it saves the outputs to a new ConfigMap which is named `$RESOURCE_NAME-output`.  In this ConfigMap, each Terraform output is a data key. 

## Custom Terraform Runner

The user can opt to use their own image for the Terraform Runner instead of the default `isaaguilar/tfops` image. The image is configurable in `spec.terraformRunner`.  There are many reason a user may opt this option:

1) Maybe the user has a narrow set of 3rd party images available. One option is to build the images themselves. The user can run the following (from the root of this repo):

```bash
# build
DOCKER_REPO=myrepo make docker-build-job
# push
DOCKER_REPO=myrepo make docker-push-job
```

This will build all the Terraform versions >=0.11.8 using the same scripts as the default Terraform Runner image. 

2) Perhaps there are issues with [`run.sh`](../docker/terraform/run.sh) that the user wants to add/modify/remove. The user can update the `run.sh` script and then build using the same command as above:

```bash
# build
DOCKER_REPO=myrepo make docker-build-job
# push
DOCKER_REPO=myrepo make docker-push-job
```

Or contribute improvements to `run.sh` in a pull-request; always appreciated. 