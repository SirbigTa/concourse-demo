Concourse Demo
==============

Prerequisites
-------------

* Docker engine
* Docker Compose


Spin up local Concourse with Docker Compose
-------------------------------------------

```bash
docker compose up -d
```

Afterwards open [http://localhost:8080](http://localhost:8080) and login with user `test` and password `test`.

**Caution** do not use this setup in production, since we are using non-secret ssh keys which are checked-in to this repository.


1st Tutorial: Download `fly`cli and login to Concourse
------------------------------------------------------

Download `fly` CLI with

```bash
export concourse_fqdn=localhost
curl --noproxy ${concourse_fqdn} -s -f -o /usr/local/bin/fly "http://${concourse_fqdn}:8080/api/v1/cli?arch=amd64&platform=darwin"
chmod u+x /usr/local/bin/fly
```

Login to Concourse via

```bash
fly --target=demo login \
    --concourse-url="http://${concourse_fqdn}:8080" \
    --username=test \
    --password=test \
    --team-name=main
```

this will generate a fly target `demo` that you have to reference in your further fly operations, e.g.

```bash
fly -t demo pipelines
```

> Normally `/usr/local/bin` would be a good place to download the fly cli locally. Since we are working with multiple Concourse versions in parallel, we put the corresponding fly binary into the root of the project folder and reference this one from within all our scripts in the project. An alternative would be the usage of the [`fly sync`](https://concourse-ci.org/fly.html#fly-sync) command.


2nd Tutorial: Upload your first Pipeline
----------------------------------------

Our first pipeline contains only a single job with a single task that outputs "Hello World":

```yaml
---
jobs:
  - name: hello-world
    plan:
      - task: hello-world
        config:
          platform: linux
          image_resource:
            type: docker-image
            source: {repository: busybox}
          run:
            path: echo
            args:
              - hello world
```

we use `fly`'s `set-pipeline` command to upload the pipeline to Concourse:

```bash
cd tutorial-2
fly --target=demo set-pipeline \
    --non-interactive \
    --pipeline=tutorial-2 \
    --config=pipeline.yml
```

afterwards we *unpause* the pipeline via

```bash
fly --target=demo unpause-pipeline -p tutorial-2
```

Now go to the Concourse UI in your browser again and after login, you will see a first pipeline. You can directly access it via [http://localhost:8080/teams/main/pipelines/tutorial-2](http://localhost:8080/teams/main/pipelines/tutorial-2)


3rd Tutorial: Create a Pipeline with an Input
---------------------------------------------

We will know create a pipeline that uses the content of a git repository as an input for a task. For simplicity, we will use this repository and run a simple shell script within it.

Here's our pipeline:

```yaml
---
resources:
  - name: demo-repo
    type: git
    source:
      uri: https://github.com/michaellihs/concourse-demo.git
      branch: master

jobs:
  - name: hello-world
    plan:
      - get: demo-repo
      - task: hello-world
        image: busy-box
        file: demo-repo/tutorial-3/hello-world.yml
 
```

here is our `task` file:

```yaml
---
platform: linux

image_resource:
  type: docker-image
  source: {repository: busybox}

inputs:
  - name: demo-repo

run:
  path: demo-repo/tutorial-3/hello-world.sh
```

We use the very same commands as in tutorial-2 to set up our pipeline.


### Bonus Round: Run Tasks directly

Sometimes, when you debug your pipeline, you want to run only single tasks until they do what you want them to do. Therefore it can be helpful run the task "stand-alone", i.e. without making many debug-commits. The following `fly` command can help you here (requires previous login to Concourse):

```bash
cd tutorial-3
fly -t demo e -c hello-world.yml -i demo-repo=../
```


4th Tutorial: Parametrized Tasks
--------------------------------

Parameters in Concourse pipelines are "injected" into your task scripts as environment variables.

Here is a parametrized pipeline version of our previous tutorial (mind the `params` section in the `task` config):

```yaml
---
resources:
  - name: demo-repo
    type: git
    source:
      uri: https://github.com/michaellihs/concourse-demo.git
      branch: master
  - name: busy-box
    type: docker-image
    source: {repository: busybox}

jobs:
  - name: hello-world
    plan:
      - get: demo-repo
      - get: busy-box
      - task: hello-world
        image: busy-box
        file: demo-repo/tutorial-4/hello-params.yml
        params:
          GREETING: "hello parameters"
```

here's the task yaml:

```yaml
---
platform: linux

inputs:
  - name: demo-repo

params:
  GREETING:

run:
  path: demo-repo/tutorial-4/hello-params.sh
```

Within your shell script, you can access the `GREETING` parameter as an env var:

```bash
#!/bin/sh

echo "${GREETING}"
```

We use the very same commands like in the previous tutorials to setup the pipeline.

> You might not immediately see the benefit of externalizing tasks - but as one side effect, this allows you to re-use tasks across multiple pipeline and finally create libraries for your pipeline tasks.


5th Tutorial: Use Variables and Settings
----------------------------------------

In comparison to parameters shown in the previous tutorial, variables are placeholders in your pipeline yaml files, that are filled when you upload your pipeline to concourse via `fly`. Apply a variable by using the `((VARIABLE_NAME))` syntax:

```yaml
---
resources:
  - name: demo-repo
    type: git
    source:
      uri: https://github.com/michaellihs/concourse-demo.git
      branch: master
  - name: busy-box
    type: docker-image
    source: {repository: busybox}

jobs:
  - name: hello-settings
    plan:
      - get: demo-repo
      - get: busy-box
      - task: hello-parameters
        image: busy-box
        file: demo-repo/tutorial-5/hello-settings.yml
        params:
          GREETING: ((greeting))
```

You now have 2 choices to fill this placeholder:

1. by using `fly`'s ` --var=[NAME=STRING]` option
2. by creating another yaml file with key value pairs and reference it via `fly`'s `--load-vars-from=` option

we go with the second option and create a file `settings.yml` with the following content

```yaml
greeting: "hello settings"
```

we now use the modified `fly` command to upload the pipeline:

```bash
fly --target=demo set-pipeline \
    --non-interactive \
    --pipeline=tutorial-5 \
    --load-vars-from=settings.yml \
    --config=pipeline.yml
```

> You can use the `--load-vars-from=` option multiple times and use it to build a chain of overrides for you placeholders, e.g. to create Ansible-style settings override for more complex environments where you merge instance-specific settings over stage-specific settings over global settings...


6th Tutorial: Use Credentials from Vault
----------------------------------------



7th Tutorial: Building a Docker Image
-------------------------------------



8th Tutorial: Use Meta Pipeline
-------------------------------

As the *Meta Pipeline* we denote a pipeline that sets up other pipelines in Concourse. In this case, the meta pipeline in tutorial 8 creates all the pipelines within this demo repository besides the pipeline for tutorial 8 itself.

To set up the pipeline, run

```bash
cd tutorial-8
./setup-pipeline.sh
```

To make sure that the pipeline really worked, delete all pipelines (but the meta pipeline) via

```bash
for i in $(seq 1 7); do
    fly -t demo destroy-pipeline -p tutorial-${i}
done
```