# Dependent Jobs

:::{admonition} Overview
:class: note
**Teaching:** 5 min | **Exercises:** 5 min

**Questions**
- How do you make some jobs run after other jobs?

**Objectives**
- Run some jobs in serial.
:::

## Defining dependencies

From the last session, we're starting with

```yaml
jobs:
  build_skim:
    runs-on: ubuntu-latest
    container: rootproject/root:${{ matrix.version }}
    strategy:
      matrix:
        version: [6.32.04-ubuntu24.04, latest]
    steps:
      - name: checkout repository
        uses: actions/checkout@v6

      - name: build
        run: |
          COMPILER=$(root-config --cxx)
          FLAGS=$(root-config --cflags --libs)
          $COMPILER -g -O3 -Wall -Wextra -Wpedantic -o skim skim.cxx $FLAGS
```

We're going to talk about another useful parameter `needs`.

:::{admonition} Specify dependencies between jobs
:class: tip
The key-value `needs: job or list of jobs` allows you to specify dependencies between jobs in the order you define.
<br/>Example:
```yaml
job2:
  needs: job1
```
job2 waits until job1 completes successfully. [Further reading](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idneeds).
:::

## A Skimmer Higgs

Let's just attempt to try and get the code working as it is. Since it builds, surely the CI/CD must be able to run it, right? 

We need to add a job named `skim`. Let's go ahead and figure out how to define a run job.

The executable `skim` takes 5 arguments: input (remote data), output, cross-section, integrated luminosity, and scale.
Let's consider the following values
```yaml
input: root://eospublic.cern.ch//eos/root-eos/HiggsTauTauReduced/GluGluToHToTauTau.root
output: skim_ggH.root
cross_section: 19.6
integrated_luminosity: 11467.0
scale: 0.1
```

The skim job should look like this:
```yaml
  skim:
    needs: build_skim
    runs-on: ubuntu-latest
    container: rootproject/root:6.32.04-ubuntu24.04
    steps:
      - name: install XRootD client
        run: apt-get update && apt-get install -y xrootd-client
      - name: skim
        run: |
          chmod +x ./skim
          ./skim root://eospublic.cern.ch//eos/root-eos/HiggsTauTauReduced/GluGluToHToTauTau.root skim_ggH.root 19.6 11467.0 0.1
```

:::{admonition} Remote `root://` files need the XRootD client
:class: caution
The input file is not part of our repository: `skim` streams it from CERN's EOS public storage over the XRootD protocol 
(that's what the `root://eospublic.cern.ch//…` URL means). 

The Ubuntu-based `rootproject/root` images ship ROOT's XRootD plugin, 
but not the underlying XRootD client library. Without it, opening any `root://` URL crashes 

The extra `install XRootD client` step above installs the 
`xrootd-client` package, which provides that library (plus handy command-line tools like `xrdcp`). 
:::

::::{admonition} Did it work?
:class: important
Let's have a look at the log message
:::{admonition} Result
:class: dropdown
```text
./skim: not found
```
:::
::::

## Artifacts

Ok, fine. That was way too easy. It seems we have a few issues to deal with. The `skim` binary in the `build_skim` job isn't in the `skim` job by default. 
We need to use GitHub `artifacts` to copy over this from the right job.

:::{admonition} Passing data between two jobs in a workflow
:class: tip

Artifacts are used to upload (`upload-artifact`) and download  (`download-artifact`) files and directories which should be attached to the job after this one has completed. 
That way it can share those files with another job in the same workflow.

```yaml
job_1:
  - uses: actions/upload-artifact@v7
    with:
      name: <name>
      path: <file>
job_2:
  - uses: actions/download-artifact@v8
    with:
      name: <name>
```
:::


:::{admonition} More Reading
:class: seealso
- [https://docs.github.com/en/actions/tutorials/store-and-share-data](https://docs.github.com/en/actions/tutorials/store-and-share-data)
:::

**Note** that the artifact name should not contain any of the following characters `"`,`:`,`<`,`>`,`|`,`*`,`?`,`\`,`/`.

Let's pass the `skim` binary from the `build_skim` job to the `skim` job using artifacts:
```yaml
name: example
on: push

jobs:
  build_skim:
    runs-on: ubuntu-latest
    container: rootproject/root:${{ matrix.version }}
    strategy:
      matrix:
        version: [6.32.04-ubuntu24.04, latest]
    steps:
      - name: checkout repository
        uses: actions/checkout@v6
      - name: build
        run: |
          COMPILER=$(root-config --cxx)
          FLAGS=$(root-config --cflags --libs)
          $COMPILER -g -O3 -Wall -Wextra -Wpedantic -o skim skim.cxx $FLAGS
      - uses: actions/upload-artifact@v7
        with:
          name: skim${{ matrix.version }}
          path: skim
          
  skim:
    needs: build_skim
    runs-on: ubuntu-latest
    container: rootproject/root:6.32.04-ubuntu24.04
    steps:
      - uses: actions/download-artifact@v8
        with:
          name: skim6.32.04-ubuntu24.04
      - name: install XRootD client
        run: apt-get update && apt-get install -y xrootd-client
      - name: skim
        run: |
          chmod +x ./skim
          ./skim root://eospublic.cern.ch//eos/root-eos/HiggsTauTauReduced/GluGluToHToTauTau.root skim_ggH.root 19.6 11467.0 0.1

```



:::{admonition} Artifact names must be unique
:class: caution
Artifacts are immutable: each artifact name can only be uploaded once per workflow run. 
A matrix job runs the same steps once per matrix entry, so each entry has to upload its artifact under a different name,
which is why we template the artifact name with `${{ matrix.version }}` below. The download step must then 
request that exact name.
:::


:::{admonition} Key Points
:class: note
- We can specify dependencies between jobs running in a series using the needs value.
- Artifacts are files created by the CI that are offered for download and inspection.

:::
