# Homework

:::{admonition} Overview
:class: note
**Exercises:** 30 min

**Questions**
- Where to go next?
- Want some homework?

**Objectives**
- Add more testing, perhaps to statistics.
:::

Like the last section, I will simply explain what you need to do. After the previous section, you should have the following in `.github/workflows/main.yml`:

```yaml
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

      - name: skim
        run: |
          chmod +x ./skim
          ./skim root://eospublic.cern.ch//eos/root-eos/HiggsTauTauReduced/GluGluToHToTauTau.root skim_ggH.root 19.6 11467.0 0.1 > skim_ggH.log

      - uses: actions/upload-artifact@v7
        with:
          name: skim_ggH
          path: |
            skim_ggH.root
            skim_ggH.log
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

  test:
    needs: [skim, plot]
    runs-on: ubuntu-latest
    container: rootproject/root:6.32.04-ubuntu24.04
    steps:
      - name: checkout repository
        uses: actions/checkout@v6

      - name: Download from skim
        uses: actions/download-artifact@v8
        with:
          name: skim_ggH

      - name: Download from plot
        uses: actions/download-artifact@v8
        with:
          name: histograms

      - name: cutflow test
        run: python3 tests/test_cutflow_ggH.py

      - name: plot test
        run: python3 tests/test_plot_ggH.py
```

In your `virtual-pipelines-eventselection` repository, you need to:

1. Add more tests
2. Go wild!

:::{admonition} Key Points
:class: note
- Use everything you've learned to write your own CI/CD!
:::
