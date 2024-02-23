
# Tekton pipeline to build and deploy Serverless Workflows
* Builds the containerized image of a given serverless workflow (identified by git URL)
* Generates the deployment manifests and a base kustomize project to install it and pushes them on a new branch of the 
  source repository

Actual deployment requires manual operation to define the configuration variables and to install using `kustomize`

## Default assumptions
* The image is generated and pushed to `quay.io/orchestrator` and this is not configurable at the moment
* The image name has a fixed, non configurable prefix equal to `serverless-workflow-`
* The target namespace of both the pipeline and the workflow is not configurable and equal to `sonataflow-infra`

## Prerequisites
* Install `RH OpenShift Pipelines` operator on all the namespaces
* For each workflow being built, create an empty repository named `serverless-workflow-<WORKFLOW-ID>` under the 
  `quay.io/orchestrator` organization

## Preliminary steps
### Installing docker credentials
To allow pushing to the registry, we need an account capable to push the image to the registry:

* Create or edit a [Robot account](https://quay.io/organization/orchestrator?tab=robots) and grant it `Write` permissions 
  to the newly created repository
* Download the `Docker configuration` file for the robot account and move it under the root folder of this repository (we 
  assume the file name is `orchestrator-orchestror-ci-auth.json`)
* Run the following to create the `docker-credentials` secret:
```bash
oc create secret -n sonataflow-infra generic docker-credentials --from-file=config.json=orchestrator-orchestror-ci-auth.json
```

### Define the SSH credentials
The pipeline uses SSH to push the deployment configuration to the source repository.

Follow these steps to properly configure the credentials in the namespace:
* Generate default SSH keys under the `ssh` folder
```bash
ssh-keygen -t rsa -b 4096 -f ssh/id_rsa -N "" -C git@github.com -q
```
* Add the SSH key to your GitHub account using the `gh` CLI or using the [SSH keys](https://github.com/settings/keys) service:
```bash
gh ssh-key add ssh/id_rsa.pub --title "Tekton pipeline"
```
* Create a `known_hosts` file by scanning the 
```bash
ssh-keyscan github.com > ssh/known_hosts
```
* Create the secret used by the Pipeline to store the SSH credentials:
```bash
oc create secret -n sonataflow-infra generic git-ssh-credentials \
  --from-file=ssh/id_rsa \
  --from-file=ssh/config \
  --from-file=ssh/known_hosts
```

**Note**: if you change the SSH key type from the default value `rsa`, you need to update the last command and also the content
of the [config](./ssh/config) file

## Pipeline parameters
| Name | Description | Default |
|======|=============|=========|
|`gitUrl`|The https URL of the repository to clone|**provided**|
|`sshGitUrl`|The SSH URL of the repository to clone|**provided**|
|`workflowId`|The workflow ID from the repository (must match the folder name)|**provided**|
|`convertToFlat`|Whether conversion to flat layout is needed or it's already flattened|`true`|

You can configure the execution parameters by changing the content of [run.properties](./kustomize/run/run.properties).

## Tekton Hub Tasks
The following tasks are automatically installed and oatched by the installation procedure:
* [git cli](https://hub.tekton.dev/tekton/task/git-cli)
* [git clone](https://hub.tekton.dev/tekton/task/git-clone)
* [kaniko](https://hub.tekton.dev/tekton/task/kaniko)

## Install
Deploy the required Tekton tasks and pipeline:
```bash
kustomize build kustomize/base | oc apply -f -
```

Run the pipeline for a specific configuration, as defined in [run.properties](./kustomize/run/run.properties):
```bash
cd kustomize/run
kustomize edit set namesuffix $(( RANDOM % 100 ))
cd ../..
kustomize build kustomize/run | oc apply -f -
```
The above will create the `PipelineRun` with a random name because these instances are immutable and cannot be updated after they were created.

## Uninstall
Delete all the Tekton esources:
```bash
kustomize build kustomize/run | oc delete -f -
kustomize build kustomize/base | oc delete -f -
```

Delete all the required secrets:
```bash
oc delete -n sonataflow-infra secret generic docker-credentials
oc delete -n sonataflow-infra secret generic git-ssh-credentials
```