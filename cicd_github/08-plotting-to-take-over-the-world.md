# Making Plots

:::{admonition} Overview
:class: note
**Teaching:** 5 min | **Exercises:** 10 min

**Questions**
- How do we make plots?

**Objectives**
- Use everything you learned to make plots!
:::

## On Your Own

So in order to make plots, we just need to take the skimmed file `skim_ggH.root` and pass it through the `histograms.py` code that exists. This can be run with the following code

```bash
python3 histograms.py skim_ggH.root ggH hist_ggH.root
```

This needs to be added to your `.github/workflows/main.yml`. 

::::{admonition} Adding Artifacts
:class: important
So we need to do two things:

1. add a `plot` job
2. save the output `hist_ggH.root` as an artifact

You know what? While you're at it, why not drop the version matrix from `build_skim` as well? We only need the pinned ROOT version from here on 🙂.

:::{admonition} Solution
:class: dropdown
```yaml
name: example
on: push

jobs:
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

      - uses: actions/upload-artifact@v7
        with:
          name: skim
          path: skim

  skim:
    needs: build_skim
    runs-on: ubuntu-latest
    container: rootproject/root:6.32.04-ubuntu24.04
    steps:
      - name: checkout repository
        uses: actions/checkout@v6
        
      - uses: actions/download-artifact@v8
        with:
          name: skim
          
      - name: install XRootD client
        run: apt-get update && apt-get install -y xrootd-client
        
      - name: skim
        run: |
          chmod +x ./skim
          ./skim root://eospublic.cern.ch//eos/root-eos/HiggsTauTauReduced/GluGluToHToTauTau.root skim_ggH.root 19.6 11467.0 0.1

      - uses: actions/upload-artifact@v7
        with:
          name: skim_ggH
          path: skim_ggH.root
          retention-days: 7

  plot:
    needs: skim
    runs-on: ubuntu-latest
    container: rootproject/root:6.32.04-ubuntu24.04
    steps:
      - name: checkout repository
        uses: actions/checkout@v6

      - uses: actions/download-artifact@v8
        with:
          name: skim_ggH

      - name: plot
        run: python3 histograms.py skim_ggH.root ggH hist_ggH.root

      - uses: actions/upload-artifact@v7
        with:
          name: histograms
          path: hist_ggH.root
```

Since the matrix is gone, the build artifact no longer needs a per-version name — we renamed it to plain `skim` (and the download step in the `skim` job must match).
:::
::::


Once we're done, we should probably start thinking about how to test some of these outputs we've made. We now have a skimmed ggH ROOT file and a file of histograms of the skimmed ggH.

:::{admonition} Are we testing anything?
:class: tip
Continuous Integration is actually testing that the scripts we have still run. So we are constantly testing as we go here which is nice. 

Additionally, there's also continuous deployment because we've been making artifacts that are passed to other jobs. 
:::

:::{admonition} Key Points
:class: note
- Another action, another job, another artifact.
:::
