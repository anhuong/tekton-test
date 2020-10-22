# Tekton Learning

## Pre-Requisite

### Setup Local Env
- Create test cluster (IBM Cloud Kube Cluster or minikube)
  - Connect to cluster -- `ic ks cluster config --cluster <name>`
- Download tekton-cli -- https://github.com/tektoncd/cli
  - See useful commands https://github.com/tektoncd/cli#useful-commands
  ```sh
  $ brew tap tektoncd/tools
  $ brew install tektoncd/tools/tektoncd-cli
  ```

## Background Info on Tekton
- Tekton - native Kube resources for building and executing CI/CD pipelines
  - successor/built from Knative build-pipelines.
  - Positioned to be the first open standard for defining native Kube CI/CD pipelines on any platform (Cloud provider, CD tool).
- Set of yamls define the pipeline but the yamls are meant to be simple to abstract away the implementation details to allow for flexibility.
- Relies on containers as building blocks to execute commands, similar to knative build-pipeline.

Comprised of CRDs…
- `Task` — sequence of steps (individual commands) that initialize init containers and commands to execute —> Pods
    - inputs/outputs (PipelineResources) need to be stored in common location — PVC or COS
    - discrete, self contained part of a process
    - Steps —> Containers
- `TaskRun`  — execution of `Task`, binds PipelineResource to Task, auto-created when create PipelineRun
    - managed pod, deploying as many containers as steps
- `PipelineResources` — I/O for Pipeline Tasks, ie git repo, container image, storage
- `Pipeline` — collection of `PipelineResources` and `Tasks`
    - ordered set of Tasks
- `PipelineRun`  — execution of `Pipeline`, binds Pipeline, Tasks, and PipelineResources —> instance of Pipeline
    - creates pod for each task
- PipelineRun --> Pipeline --> TaskRun --> PipelineResource + Task

Super useful doc on Tekton specs and all the yaml pieces and fields. https://github.com/tektoncd/pipeline/tree/master/docs

NOTE: names must all follow these [kube object dns guidelines](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names).
  - only lowercase alphanumeric characters or `-`
  - start and end with alphanumeric character

NOTE: when you deploy a directory, it deploys in the order the files exist.

### Task

Task = sequence of `steps` each with own container.
Comprised of:
  - `params` with name, type, description, default.
    - `type` can be `string` or `array`.
    - Tekton variable substition via `$(params.myParamName)` or with array `$(params.arrayParam[*])`. Arrays must be referenced in their entirety and completely isolated. See notes [here](https://github.com/tektoncd/pipeline/blob/master/docs/tasks.md#substituting-array-parameters).
    - Params are from the TaskRun with arrays being a yaml array.

  - `resources` = expected inputs and output resources with own `PipelineResource` and then connected via TaskRun.

  - `steps` - reference container image to run and what to run.
    - name, image, env, command/script, and args.
    - Can either have a specific command (like echo) or run a script where you can specify the shebang at the top to dictate the language that is run. Your script can also run other scripts of code.

  - `workspaces` - specify 1+ volumes.
    - name, description, mountPath

  - `results` - string results that can be emitted to user to passed to other Tasks in Pipeline.
    - Each result entry corresponds to a file stored in /tekton/results (aka `$(results.<name>.path)`).
    - A Pipeline can pass the result of a Task into the `Parameters` or `WhenExpressions` of another Task.
    - A Pipeline can itself emit results and include data from the results of its Tasks.
      - Add a `results` section to the Pipeline with name, description, and value and for the value use variable substitution `$(tasks.<task-name-in-pipline>.results.<result-name-in-task>)`


  - reserved directories:
    - `/workspace` - for resources and workspaces to mount.
    - `/tekton` - Tekton specific functionaltiy like `/tekton/results` is where results are written to.

### Pipeline

## Tutorial

Tekton Tutorial
https://github.com/tektoncd/pipeline/blob/master/docs/tutorial.md

### Simple Example
1. Install tekton pipeline and dependencies on cluster:
```sh
  $ kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

  # verify
  $ kubectl get pods --namespace tekton-pipelines
  NAME                                           READY   STATUS    RESTARTS   AGE
tekton-pipelines-controller-64d7744d48-zrg25   1/1     Running   0          15m
tekton-pipelines-webhook-f4fb59f4d-2v2pm       1/1     Running   0          15m

```

2. Apply `Task`

Task defines a series of `steps` run in a certain order.
Each task runs in a Pod, each step on a container.

```
  $ kubectl apply -f task.yaml
  task.tekton.dev/echo-hello-world created

  # verify
  $ tkn task list
  NAME               DESCRIPTION   AGE
  echo-hello-world                 6 minutes ago

  $ tkn task describe echo-hello-world
  Name:        echo-hello-world
  Namespace:   default

  📨 Input Resources
   No input resources

  📡 Output Resources
   No output resources

  ⚓ Params
   No params

  🦶 Step
   ∙ echo

  🗂  Taskruns
   No taskruns

  $ tkn task logs echo-hello-world
  Error: no TaskRuns found for Task echo-hello-world
```

3. Instantiate `Task` using `TaskRun`
```sh
  $ kubectl apply -f task-run.yaml
  taskrun.tekton.dev/echo-hello-world-task-run created

  # verify
  $ tkn taskrun list
  NAME                        STARTED          DURATION     STATUS
  echo-hello-world-task-run   20 seconds ago   11 seconds   Succeeded

  $ tkn taskrun describe echo-hello-world-task-run
  Name:        echo-hello-world-task-run
  Namespace:   default
  Task Ref:    echo-hello-world
  Timeout:     1h0m0s
  Labels:
   app.kubernetes.io/managed-by=tekton-pipelines
   tekton.dev/task=echo-hello-world

  🌡️  Status
  STARTED          DURATION     STATUS
  27 seconds ago   11 seconds   Succeeded

  📨 Input Resources
   No input resources

  📡 Output Resources
   No output resources

  ⚓ Params
   No params

  🦶 Steps
   NAME     STATUS
   ∙ echo   Completed

  🚗 Sidecars
  No sidecars

  # two ways to see logs
  $ tkn taskrun logs echo-hello-world-task-run
  [echo] Hello World

  $ tkn task logs echo-hello-world
  [echo] Hello World

```

Similar to all kube objects, if you redeploy a tekton resource, it will see if anything has changed and reconfigure the kube object.
```sh
  $ kubectl apply -f task-run.yaml
  taskrun.tekton.dev/echo-hello-world-task-run configured
  taskrun.tekton.dev/echo-pi-task-run created
```

I tried editting the echo-hello-world Task to include a third step. I redeployed it using `kubectl apply`. And I see that it was reconfigured and updated.
```sh
# Although Task was updated, the TaskRun had already run
$ tkn task describe echo-hello-world
Name:        echo-hello-world
Namespace:   default

📨 Input Resources

 No input resources

📡 Output Resources

 No output resources

⚓ Params

 No params

🦶 Steps

 ∙ echo
 ∙ echo-again
 ∙ third-step

🗂  Taskruns

NAME                        STARTED         DURATION     STATUS
echo-hello-world-task-run   6 minutes ago   44 seconds   Succeeded
```

BUT the TaskRun did not run again so the logs only had two echo messages. I redeployed the TaskRun but it did not change anything and did not retrigger it to run. So to cause another TaskRun to run, I ran `tkn task start`
```sh
# Trigger another TaskRun
$ tkn task start echo-hello-world
TaskRun started: echo-hello-world-run-lq59t

In order to track the TaskRun progress run:
tkn taskrun logs echo-hello-world-run-lq59t -f -n default

$ tkn taskruns list
NAME                         STARTED          DURATION     STATUS
echo-hello-world-run-lq59t   13 seconds ago   9 seconds    Succeeded
echo-hello-world-task-run    8 minutes ago    44 seconds   Succeeded
echo-pi-task-run             8 minutes ago    45 seconds   Succeeded

# Verify
$ tkn taskrun logs echo-hello-world-run-lq59t
[echo] Hello World

[echo-again] Another echo message

[third-step] Third echo message

# See that there are two different TaskRuns associated with single Task
$ tkn task describe echo-hello-world
Name:        echo-hello-world
Namespace:   default

📨 Input Resources

 No input resources

📡 Output Resources

 No output resources

⚓ Params

 No params

🦶 Steps

 ∙ echo
 ∙ echo-again
 ∙ third-step

🗂  Taskruns

NAME                         STARTED          DURATION     STATUS
echo-hello-world-run-lq59t   2 minutes ago    9 seconds    Succeeded
echo-hello-world-task-run    10 minutes ago   44 seconds   Succeeded

# When looking at Tasks logs, must select which of the TaskRuns want logs from.
$ tkn task logs echo-hello-world
? Select taskrun: echo-hello-world-run-lq59t started 2 minutes ago
[echo] Hello World

[echo-again] Another echo message

[third-step] Third echo message
```

Also works well if delete the taskruns and redeploy or delete all and redeploy.

4. Cleanup workspace
```sh
$ tkn task delete --all --force
All Tasks deleted in namespace "default"

$ tkn taskrun delete --all --force
All TaskRuns deleted in namespace "default"
```

<br>

### Input/Outputs - PipelineResource

https://github.com/tektoncd/pipeline/blob/master/docs/resources.md
Types of resources are: `git`, `pullRequest`, `image`, `cluster`, `storage`, and `cloudevent`.

1. Create `PipelineResource` for inputs/outputs.
  Use one or more `PipelineResource` to define the artifacts you want to pass in and out of your `Task`.

  In our example, we have a `git` PipelineResource which is the input of the source code. And an `image` PipelineResource output.

2. Create `Task` that accepts given inputs and outputs.

  Task specifies what is happening -- building an image using the given input source code and will output the image of given name.

  Task accepts `spec.resources.inputs` and `spec.resources.outputs`. These can be used as params for the `steps` in the Task.
  `$(resources.inputs.docker-source.path)` and `$(resources.outputs.builtImage.url)`.

  Question in example, in steps, they reference the `$(params.pathToContext)` in the `steps` and `$(resources.outputs.builtImage.url)`. Instead of the params, could they have just referenced the `resources.inputs` directly? Or did it have to be passed in as a param for the steps to use?

3. Create Secret and ServiceAccount to push image to docker registry.

```sh
$ kubectl create secret docker-registry anh-uong-iam-api-key-redsonja --docker-server="https://us.icr.io" --docker-username=iamapikey --docker-password=<my-secret> --docker-email="anh.uong@ibm.com"

secret/anh-uong-iam-api-key-redsonja created

$ kubectl apply -f docker-secret-service-account.yaml
serviceaccount/redsonja-docker-registry-secret created
```

4. Create `TaskRun` that bind the specific `PipelineResource` to the `Task` and sets values for variable substitution parameters.

Connects the `PipelineResource` kube objects to the `Task` object and passes in the needed parameters for the `Task`.

Question: What is the difference between inputs and parameters for Tasks?

5. Deploy everything with kubectl apply

Can view tekton deployed pieces using `kubectl get tekton-pipelines`.

```sh
$ kubectl get tekton-pipelines
NAME                                                             SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
taskrun.tekton.dev/build-docker-image-from-git-source-task-run   Unknown     Pending   7s          

NAME                                                 AGE
task.tekton.dev/build-docker-image-from-git-source   7s

NAME                                                           AGE
pipelineresource.tekton.dev/skaffold-git                       34s
pipelineresource.tekton.dev/skaffold-image-hello-world-flask   34s

$ tkn taskrun list
NAME                                          STARTED          DURATION   STATUS
build-docker-image-from-git-source-task-run   19 seconds ago   ---        Running

$ tkn task list
NAME                                 DESCRIPTION   AGE
build-docker-image-from-git-source                 25 seconds ago
```

It ran through successfully and built and pushed the image up to IBM Cloud redsonja registry.

```
$ tkn taskrun describe build-docker-image-from-git-source-task-run
Name:              build-docker-image-from-git-source-task-run
Namespace:         default
Task Ref:          build-docker-image-from-git-source
Service Account:   redsonja-docker-registry-secret
Timeout:           1h0m0s
Labels:
 app.kubernetes.io/managed-by=tekton-pipelines
 tekton.dev/task=build-docker-image-from-git-source

🌡️  Status

STARTED          DURATION    STATUS
36 seconds ago   ---         Running

📨 Input Resources

 NAME              RESOURCE REF
 ∙ docker-source   skaffold-git

📡 Output Resources

 NAME           RESOURCE REF
 ∙ builtImage   skaffold-image-hello-world-flask

⚓ Params

 NAME                 VALUE
 ∙ pathToDockerFile   Dockerfile
 ∙ pathToContext      $(resources.inputs.docker-source.path)/examples/microservices/leeroy-web

🦶 Steps

 NAME                               STATUS
 ∙ build-and-push                   Running
 ∙ image-digest-exporter-vsjwd      Running
 ∙ create-dir-builtimage-zsfbn      Completed
 ∙ git-source-docker-source-mfr4w   Completed

🚗 Sidecars

No sidecars

$ tkn taskrun logs build-docker-image-from-git-source-task-run

[git-source-docker-source-mfr4w] {"level":"info","ts":1603231387.3966246,"caller":"git/git.go:164","msg":"Successfully cloned https://github.com/GoogleContainerTools/skaffold @ b4c29e0bf68319635af40f53043e59e4854a58d3 (grafted, HEAD, origin/master) in path /workspace/docker-source"}
[git-source-docker-source-mfr4w] {"level":"info","ts":1603231387.4677374,"caller":"git/git.go:205","msg":"Successfully initialized and updated submodules in path /workspace/docker-source"}

[build-and-push] INFO[0002] Resolved base name golang:1.12.9-alpine3.10 to golang:1.12.9-alpine3.10
[build-and-push] INFO[0002] Resolved base name alpine:3.10 to alpine:3.10
[build-and-push] INFO[0002] Resolved base name golang:1.12.9-alpine3.10 to golang:1.12.9-alpine3.10
[build-and-push] INFO[0002] Resolved base name alpine:3.10 to alpine:3.10
[build-and-push] INFO[0002] Retrieving image manifest golang:1.12.9-alpine3.10
[build-and-push] INFO[0003] Retrieving image manifest golang:1.12.9-alpine3.10
[build-and-push] INFO[0005] Retrieving image manifest alpine:3.10        
[build-and-push] INFO[0005] Retrieving image manifest alpine:3.10        
[build-and-push] INFO[0006] Built cross stage deps: map[0:[/web]]        
[build-and-push] INFO[0006] Retrieving image manifest golang:1.12.9-alpine3.10
[build-and-push] INFO[0006] Retrieving image manifest golang:1.12.9-alpine3.10
[build-and-push] INFO[0007] Unpacking rootfs as cmd COPY web.go . requires it.
[build-and-push] INFO[0018] Taking snapshot of full filesystem...        
[build-and-push] INFO[0020] COPY web.go .                                
[build-and-push] INFO[0020] Taking snapshot of files...                  
[build-and-push] INFO[0020] RUN go build -o /web .                       
[build-and-push] INFO[0020] cmd: /bin/sh                                 
[build-and-push] INFO[0020] args: [-c go build -o /web .]                
[build-and-push] INFO[0022] Taking snapshot of full filesystem...        
[build-and-push] INFO[0023] Saving file /web for later use.              
[build-and-push] INFO[0023] Deleting filesystem...                       
[build-and-push] INFO[0024] Retrieving image manifest alpine:3.10        
[build-and-push] INFO[0024] Retrieving image manifest alpine:3.10        
[build-and-push] INFO[0025] Unpacking rootfs as cmd COPY --from=builder /web . requires it.
[build-and-push] INFO[0025] Taking snapshot of full filesystem...        
[build-and-push] INFO[0025] ENV GOTRACEBACK=single                       
[build-and-push] INFO[0025] CMD ["./web"]                                
[build-and-push] INFO[0025] COPY --from=builder /web .                   
[build-and-push] INFO[0025] Taking snapshot of files...                  

[image-digest-exporter-vsjwd] 2020/10/20 22:02:59 unsuccessful cred copy: ".docker" from "/tekton/creds" to "/tekton/home": unable to open destination: open /tekton/home/.docker/config.json: permission denied
[image-digest-exporter-vsjwd] {"level":"info","ts":1603231418.6717257,"logger":"fallback-logger","caller":"imagedigestexporter/main.go:59","msg":"No index.json found for: builtImage","commit":"9d4d495"}

```

I did run into a few problems deploying the Tekton tasks. First, I tried deploying the hello-world-flask image, but realized the task doesn't have access to github.ibm.com.

Then, I moved the hello-world-flask source to my personal github.com account. The clone then succeeded but then had error `Error: error resolving dockerfile path: please provide a valid path to a Dockerfile within the build context with --dockerfile`. I had to tear down the Tekton pieces and redeploy to get it going.

Finally, it worked but failed to import a dependent Python package. Error:
```
ImportError: cannot import name 'Feature' from 'setuptools' (/usr/local/lib/python3.7/site-packages/setuptools/__init__.py)

ERROR: Command errored out with exit status 1: python setup.py egg_info Check the logs for full command output.

error building image: error building stage: failed to execute command: waiting for process to exit: exit status 1
```

So then I used the given image example from the tutorial and that worked.

<br>

### Pipelines

A `Pipeline` defines an ordered series of `Task`s that you want to execute along with the corresponding inputs and outputs for each Task. You can specify whether the output of one Task is used as an input for the next Task using the `from` property.

1. Create a `Pipeline` with a defined, ordered list of `tasks`.

2. Create a `PipelineRun` that specifies what `Pipeline` it is running.

Each `PipelineRun` automatically defines a corresponding TaskRun for each Task defined in the Pipeline.

3. Deploy and view the logs.

When you deploy a PipelineRun, the Pipeline must already be deployed.
Similarly for TaskRun and Tasks.
When you deploy a Pipeline, the Tasks must already be deployed. Otherwise, you will get error `Failed(CouldntGetTask) -- Pipeline default/tutorial-simple-pipeline can't be Run; it contains Tasks that don't exist: Couldn't retrieve Task "echo-inputs": task.tekton.dev "echo-inputs" not found`.

```sh
# The TaskRuns are automatically created and deployed with the Pipeline/PipelineRun
$ kubectl get tekton-pipelines
NAME                               AGE
task.tekton.dev/echo-hello-world   13m
task.tekton.dev/echo-inputs        2m49s
task.tekton.dev/echo-pi            13m

NAME                                                  SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
pipelinerun.tekton.dev/tutorial-simple-pipeline-run   Unknown     Running   7s          

NAME                                                                     SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
taskrun.tekton.dev/tutorial-simple-pipeline-run-task-echo-hello-world-xhj9l   Unknown     Running   9s          
taskrun.tekton.dev/tutorial-simple-pipeline-run-task-echo-inputs-zk2lr        Unknown     Pending   9s          
taskrun.tekton.dev/tutorial-simple-pipeline-run-task-echo-pi-pwvkc            Unknown     Running   8s          

NAME                                           AGE
pipeline.tekton.dev/tutorial-simple-pipeline   16m


# Pipline has tasks defined and attached PipelineRun
$ tkn pipeline describe tutorial-simple-pipeline
Name:        tutorial-simple-pipeline
Namespace:   default

📦 Resources

 No resources

⚓ Params

 No params

🗒  Tasks

NAME                      TASKREF            RUNAFTER   TIMEOUT   CONDITIONS   PARAMS
∙ task-echo-inputs        echo-inputs                   ---       ---          textToPrint: This is some string to print from a parameter., arrayToPrint: [ I am, an array, Hooray! ]
∙ task-echo-hello-world   echo-hello-world              ---       ---          printResult:
∙ task-echo-pi            echo-pi                       ---       ---          ---

⛩  PipelineRuns

 NAME                             STARTED        DURATION     STATUS
 ∙ tutorial-simple-pipeline-run   1 minute ago   18 seconds   Succeeded


# PipelineRun shows associated TaskRuns
$ tkn pipelinerun describe tutorial-simple-pipeline-run
Name:              tutorial-simple-pipeline-run
Namespace:         default
Pipeline Ref:      tutorial-simple-pipeline
Service Account:   default
Timeout:           1h0m0s
Labels:
tekton.dev/pipeline=tutorial-simple-pipeline

🌡️  Status

STARTED        DURATION     STATUS
1 minute ago   18 seconds   Succeeded

📦 Resources

No resources

⚓ Params

No params

🗂  Taskruns

NAME                                                    TASK NAME          STARTED        DURATION     STATUS
∙ tutorial-simple-pipeline-run-echo-pi-pwvkc            echo-pi            1 minute ago   17 seconds   Succeeded
∙ tutorial-simple-pipeline-run-echo-hello-world-xhj9l   echo-hello-world   1 minute ago   11 seconds   Succeeded
∙ tutorial-simple-pipeline-run-echo-inputs-zk2lr        echo-inputs        1 minute ago   14 seconds   Succeeded

# Output from all three tasks including one task that has a result that is passed to the next task
$ tkn pipelinerun logs || tkn pipeline logs tutorial-simple-pipeline

[task-echo-inputs : echo-array-to-print] ARRAY PARAM I am an array Hooray!

[task-echo-inputs : echo-text-to-print] TEXT PARAM
[task-echo-inputs : echo-text-to-print] This is some string to print from a parameter.

[task-echo-inputs : print-date-human-readable] Thu Oct 22 18:51:07 UTC 2020

[task-echo-hello-world : echo] Hello World

[task-echo-hello-world : echo-param] Thu Oct 22 18:51:07 UTC 2020
[task-echo-hello-world : echo-param]

[echo-pi : pi] 3.141592653589793238......
```

<br>
