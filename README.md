# Run Tekton using kind

What is kind? 

https://kind.sigs.k8s.io

```
"kind is a tool for running local Kubernetes clusters using Docker container “nodes”.
kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI."
```

Tekton needs a kubernetes environment to run on, it is possible to run Tekton on your computer using kind. You just need to setup a few layers:

## Install docker

https://docs.docker.com/install/

## Install kubectl

https://kubernetes.io/docs/tasks/tools/install-kubectl/

## Install kind

Follow the instructions on this page:

https://kind.sigs.k8s.io/docs/user/quick-start/#installation

Follow the instructions for your computer using the link shared above ^

Test kind is succesfully installed:

    $ kind version

## Start kind

I've used an example from the documentation and modified it to be one control plane node and one worker node:

    ./kind-example-config.yaml

To run this config with kind and create the 2 nodes:

    $ kind create cluster --config kind-example-config.yaml --name multi-node

List your kind clusters:

    $ kind get clusters

You should see your "multi-node" cluster listed in the response

Also have a look at your docker processes:

    $ docker ps

You should see the kind nodes listed as running containers:

```
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                       NAMES

2aa6d2ab6636        kindest/node:v1.17.0   "/usr/local/bin/entr…"   47 minutes ago      Up 47 minutes       127.0.0.1:32769->6443/tcp   multi-node-control-plane

2e8bee1c68e6        kindest/node:v1.17.0   "/usr/local/bin/entr…"   47 minutes ago      Up 47 minutes                                   multi-node-worker

```

## Check kubectl context

    $ kubectl config get-contexts

Your contet should have defaulted to your newly created kind cluster, but if it didn't you can switch context by using:

    $ kubectl config use-context kind-multi-node

Look for the * -> this asterix in the CURRENT column in the response indicates the current context

## Tekton

### Install Tekton Pipelines

https://github.com/tektoncd/pipeline

As mentioned in the repo, Run:

    $ kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

You can check the installation:

    $ kubectl get pods --namespace tekton-pipelines --watch

### Install Tekton Dashboard

https://github.com/tektoncd/dashboard

As mentioned in the repo, Run:

    $ kubectl apply --filename https://github.com/tektoncd/dashboard/releases/download/v0.6.0/tekton-dashboard-release.yaml

You can check the installation:

    $ kubectl get pods --namespace tekton-pipelines

You can use port forward to view the dashboard:

    $ kubectl get pods -n tekton-pipelines

    Find the pod tekton-dashboard-xxxxxxx-xxx

    $ kubectl port-forward tekton-dashboard-xxxxxxx-xxx -n tekton-pipelines -p 9097:9097

Should see response:

    Forwarding from 127.0.0.1:9097 -> 9097
    Forwarding from [::1]:9097 -> 9097

Open in browser:

http://localhost:9097


## Test Tekton

I have a repo that I setup a couplde examples, I just test some of my test from there on our new cluster:

    $ git clone git@github.com:CariZa/tekton-examples.git

    $ cd tekton-examples

    $ kubectl apply -f ./example-1-github-read/

You should see:

```
pipeline.tekton.dev/pipeline-test created
pipelineresource.tekton.dev/github-repo created
task.tekton.dev/read-task created
```

You can check the dashboard, you should have a pipeline, a task and a resource setup.

You haven't run anything yet, so there won't be any pipelines runs or task runs.

To trigger the pipeline to run use this command:

```
cat <<EOF | kubectl create -f -
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  name: pipelinerun-test-$(date +%s)
spec:
  pipelineRef:
    name: pipeline-test
  resources:
    - name: github-repo
      resourceRef:
        name: github-repo
EOF
```

Note:

PipelineRuns are uniquely named. TaskRuns as well. This snippet above uses a timestamps as prt of the naming convention.

It runs "kubectl create -f ..." so this creates the pipelinerun from the yaml provided in the snippet.

View the pipelinerun:

    $ kubectl get pipelinerun

Vew the taskrun created by the pipelinerun:

    $ kubectl get taskrun

View the pod created by the taskrun:

    $ kubectl get pod 

Then view logs using the pod name:

    $ kubectl logs pipelinerun-test-xxxxxx-pipeline-read-task-xxx-pod-xxxx --all-containers

Replace "pipelinerun-test-xxxxxx-pipeline-read-task-xxx-pod-xxxx" with your pod's name.

The logs should end with:

```
# ulmaceae

:shrug:
```

This was from the repo provided in the PipelineResource: "https://github.com/CariZa/ulmaceae"

And the Task step was to cat the README file:

```
  steps:
    - name: catreadme
      image: ubuntu
      command:
        - cat
      args:
        - "github-repo/README.md"
```

You can also take a look at the dashboard again, and click on the pipeline, the pipelinerun, then the task lgos will show and you will see in the dashboard logs:

```
# ulmaceae

:shrug:

Step completed
```



