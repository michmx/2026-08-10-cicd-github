# Understanding YAML and GitHub Actions

:::{admonition} Overview
:class: note
**Teaching:** 15 min

**Questions**
- What is the GitHub Actions specification?

**Objectives**
- Learn where to find more details about everything for the GitHub Actions.
- Understand the components of GitHub Actions YAML file.
- Try and see if the CI/CD can catch problems with our code.
:::

## GitHub Actions YAML

The GitHub Actions configurations are specified using YAML files stored in the`.github/workflows/` directory. Here is an example of a YAML file:


### Overall Structure

Every single parameter we consider for all configurations are keys under jobs. The YAML is structured using job names. For example, we can define two jobs that run in parallel (more on parallel/serial later) with different sets of parameters.

```yaml
name: <name of your workflow>

on: <event or list of events>

jobs:
  job_1:
    name: <name of the first job>
    runs-on: <type of machine to run the job on>
    steps:
      - name: <step 1>
        run: |
          <commands>
      - name: <step 2>
        run: |
          <commands>
  job_2:
    name: <name of the second job>
    runs-on: <type of machine to run the job on>
    steps:
      - name: <step 1>
        run: |
          <commands>
      - name: <step 2>
        run: |
          <commands>
```


- `name`: GitHub displays the names of your workflows on your repository's actions page. If you omit name, GitHub sets it to the YAML file name.

- `on`: **Required**. Specify the event that automatically triggers a workflow run. This example uses the push event, so that the jobs run every time someone pushes a change to the repository.<br/>
For more details, check [this link](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#on).

- `<job_id>`: Each job must have an id to associate with the job, job_1 in the above example. The key job_id is a string that is unique to the jobs object. It must start with a letter or _ and contain only alphanumeric characters, -, or _. Its value is a map of the job's configuration data.

- `runs-on`: **Required**. Each job runs in a particular type of machine (called a "runner") that we choose with this key. There are options for the major operating systems (Linux, Windows, and macOS) and different versions of them. The available options can be found [here](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idruns-on).

- `steps`: Specify sequence of tasks. A step is an individual task. It either runs shell commands (`run:`) or uses an *action* (`uses:`) — a reusable unit of code published on GitHub. Each step in a job executes on the same runner, allowing the steps in that job to share data with each other. If you do not provide a `name`, the step name will default to the text specified in the `run` command.

Specify the job(s) to be run. Jobs run in parallel by default. To run jobs sequentially, you have to define dependencies on other jobs. We'll cover this in a later section.


:::{admonition} Reference
:class: seealso
The reference guide for all GitHub Actions pipeline configurations is found at [workflow-syntax-for-github-actions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax). This contains all the different parameters you can assign to a job.
:::

## Time To Skim

### The Naive Attempt

As of right now, your `.github/workflows/main.yml` should look like

```yaml
name: example
on: push
jobs:
  greeting:
    runs-on: ubuntu-latest
    steps:
      - run: echo hello world
```

Let's go ahead and teach our CI to build our code. Let's add another job (named `build_skim`) that runs in parallel for right now, and runs the compiler `ROOT` uses.
<br/>

Let's give a try.

```bash
COMPILER=$(root-config --cxx)
FLAGS=$(root-config --cflags --libs)
$COMPILER -g -O3 -Wall -Wextra -Wpedantic -o skim skim.cxx $FLAGS
```

The compilation will result in an output binary called `skim`.

How do we change the CI in order to add a new job that compiles our code?

```yaml
name: example
on: push 
jobs:
  greeting:
    runs-on: ubuntu-latest
    steps:
      - run: echo hello world

  build_skim:
    runs-on: ubuntu-latest
    steps:
      - name: build
        run: |
          COMPILER=$(root-config --cxx)
          FLAGS=$(root-config --cflags --libs)
          $COMPILER -g -O3 -Wall -Wextra -Wpedantic -o skim skim.cxx $FLAGS
```

### Check jobs

Let's commit and push the changes we made, and let's go to GitHub to check if both jobs ran.

```bash
git add .github/workflows/main.yml
git commit -m "add build skim job"
git push origin main
```

### No root-config?

Ok, so maybe we were a little naive here. GitHub runners come pre-installed with a wide variety of software that is commonly 
needed in CI workflows (e.g. for Ubuntu 24.04 runners — which is what `ubuntu-latest` currently gives you — the list can be 
found [here](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md)). 

ROOT is not pre-installed, so we will have to add a step to install it ourselves. 
After reading the [ROOT documentation](https://root.cern/install/#run-in-a-docker-container), we find that a convenient way to run it on various systems is using a Docker container.

There are several tools that are used for containerization, like Docker, Podman, and Apptainer (formerly Singularity). 
For this tutorial you don't need to know anything about containerization. You can just think of this as the base software 
set that comes pre-installed on the system that runs your code.

We will be using the Docker images hosted at the [`rootproject/root` Docker Hub](https://hub.docker.com/r/rootproject/root). Let's start by using the image 
with tag `6.32.04-ubuntu24.04`.

```yaml
build_skim:
  runs-on: ubuntu-latest
  container: rootproject/root:6.32.04-ubuntu24.04
  steps:
    - name: build
      run: |
        COMPILER=$(root-config --cxx)
        FLAGS=$(root-config --cflags --libs)
        $COMPILER -g -O3 -Wall -Wextra -Wpedantic -o skim skim.cxx $FLAGS
```

Note the extra line `container: rootproject/root:6.32.04-ubuntu24.04` that specifies the container image that we want to use. 
Since it comes pre-packaged with ROOT, we do not need to have a step to install it. 

This image also contains other tools that 
we will need for the rest of the tutorial, including Python 3 — which we will invoke explicitly as `python3` later on.

## Using Actions 

What's that?

`error: skim.cxx: No such file or directory`

It seems the job cannot access the repository. We need to instruct GitHub actions to checkout the repository.

We will use the `actions/checkout` action to checkout the repository. 
This action checks out your repository under `$GITHUB_WORKSPACE`, so your workflow can access it.

```yaml
steps:
  - name: checkout repository
    uses: actions/checkout@v6
```

:::{admonition} Actions
In GitHub CI/CD, an Action is a reusable script that performs a specific step in your software development 
workflow, such as checking out a repository (`actions/checkout`), setting up a tool (`actions/setup-python`), 
deploying pages (`actions/deploy-pages`), etc. 

The [actions/checkout](https://github.com/actions/checkout) action checks out your repository under the workspace, 
so your workflow can access it.
:::

Let’s go ahead and tell our CI to checkout the repository:

```yaml
build_skim:
runs-on: ubuntu-latest
container: rootproject/root:6.32.04-ubuntu24.04
steps:
  - name: checkout repository
    uses: actions/checkout@v6    
  - name: build
    run: |
      COMPILER=$(root-config --cxx)
      FLAGS=$(root-config --cflags --libs)
      $COMPILER -g -O3 -Wall -Wextra -Wpedantic -o skim skim.cxx $FLAGS
```


### Ways to get software

As we saw before, GitHub pre-installs many common software packages and libraries that people might need, 
but often we need to install additional software. There are often actions we can use for this, 
like `actions/setup-python` to install python or `mamba-org/setup-micromamba` to install 
[Mamba](https://mamba.readthedocs.io) (an alternative to [Conda](https://docs.conda.io), an environment manager). 

These actions are simply repositories that contain scripts to install or perform certain actions. You can find more information
about these actions by going to github.com/\<name-of-action\>. For example, for `mamba-org/setup-micromamba` you can 
find more information at [https://github.com/mamba-org/setup-micromamba](https://github.com/mamba-org/setup-micromamba).

If we wanted to use Conda instead of Docker, our `build_skim` job would look like this:

```yaml
build_skim:
  runs-on: ubuntu-latest
  defaults:
    run:
      shell: bash -el {0}
  steps:
    - name: checkout repository
      uses: actions/checkout@v6
    - name: Install ROOT
      uses: mamba-org/setup-micromamba@v2
      with:
        environment-name: env
        create-args: root
    - name: build
      run: |
        COMPILER=$(root-config --cxx)
        FLAGS=$(root-config --cflags --libs)
        $COMPILER -g -O3 -Wall -Wextra -Wpedantic -o skim skim.cxx $FLAGS
```

### Building multiple versions

Great, so we finally got it working... Let's build both the version of the code we're testing and also test that the latest ROOT image (`rootproject/root:latest`) works with our code. Call this new job `build_skim_latest`.

::::{admonition} Adding the `build_skim_latest` job
:class: important

What does the `.github/workflows/main.yml` look like now?

:::{admonition} Solution
:class: dropdown

```yaml
jobs:
  greeting:
    runs-on: ubuntu-latest
    steps:
      - run: echo hello world

  build_skim:
    runs-on: ubuntu-latest
    container: rootproject/root:6.32.04-ubuntu24.04
    steps:
      - name: checkout repository
        uses: actions/checkout@v6

      - name: build
        run: |
          COMPILER=$(root-config --cxx)
          FLAGS=$(root-config --cflags --libs)
          $COMPILER -g -O3 -Wall -Wextra -Wpedantic -o skim skim.cxx $FLAGS

  build_skim_latest:
    runs-on: ubuntu-latest
    container: rootproject/root:latest
    steps:
      - name: checkout repository
        uses: actions/checkout@v6

      - name: latest
        run: |
          COMPILER=$(root-config --cxx)
          FLAGS=$(root-config --cflags --libs)
          $COMPILER -g -O3 -Wall -Wextra -Wpedantic -o skim skim.cxx $FLAGS
```
:::
::::


:::{admonition} Key Points
:class: note
- You should bookmark the [GitHub Actions reference](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax). You'll visit that page often.
- Steps run shell commands (`run:`) or invoke reusable actions (`uses:`); steps combine into jobs.
- Workflows are made up of one or more jobs and can be scheduled or triggered.
:::
