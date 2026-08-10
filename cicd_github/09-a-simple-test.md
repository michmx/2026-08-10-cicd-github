# Let's Actually Make A Test (For Real)

:::{admonition} Overview
:class: note
**Teaching:** 5 min | **Exercises:** 20 min

**Questions**
- How does a realistic workflow look for a physics analysis?
- How to generate reports?

**Objectives**
- Actually add a test on the output of running physics code
:::

## Why testing?

We have already seen how CI helps to ensure that our code compiles and runs as expected, and to store the output 
of the execution to manually check that the result is acceptable.

However, as a software development project scales, manual checks become increasingly time-consuming and error-prone. 
It becomes increasingly important to have a set of tests that can be run automatically 
to ensure that the output of the code is as expected. 
This is especially true when multiple people are working on the same codebase.

Testing itself is a broad topic, and we will not cover it in detail here. We will just illustrate how to run a simple test
in our CI pipeline, and how to keep track of the results. Your homework will be to explore the topic further to cope
with more complex scenarios.

## ABC of testing

Broadly speaking, there are different types of tests:

* **Unit tests** are minimal tests that check that a single function or class works as expected. 
  They are usually written by the developer of the function or class, and are run frequently during development.
* **Integration tests** are tests that check that multiple functions or classes work together as expected. 
  Also written by the developers and run frequently during development.
* **System tests** are tests that check that the whole system works as expected. 
  Usually written by the developer of the function or class, and are run less frequently during development.
* **Regression tests** protects existing features to ensure code updates did not break previously stable functionality.

Commonly, CI is used to run unit and integration tests, as they are designed to be run frequently. 

There are many frameworks to write tests, depending on the programming language. A few popular examples are:
* C++ : [Google Test](https://google.github.io/googletest/)
* Python: [Pytest](https://docs.pytest.org/en/latest/)
* Java: [JUnit](https://junit.org/junit5/)
* Javascript: [Jest](https://jestjs.io/)
* ...

No matter the framework, the general idea is the same: write a test that checks that a function or class works as 
expected (like checking the output), and run it. 
If the test fails, then the function or class is broken, and needs to be fixed.

Some ideas on what tests to automate in HEP analysis:
* Check that the output histogram is compatible with a reference histogram.
* Do a simple cut-and-count analysis, and check that the number of events is acceptable.
* Perform a fit, and check that the fit parameters are within a reasonable range.
Such tasks must be performed in a small subset of the data or simulation, so that the tests can be run frequently.  

## Pytest

We will use [pytest](https://docs.pytest.org/), the most widely used
testing framework for Python. The idea is simple: you write small functions that check one thing each, and
pytest finds them, runs them, and reports which ones passed or failed.

It can be installed with pip:

```bash
pip install pytest
```

### How pytest finds your tests

You don't register tests anywhere, pytest discovers them by naming convention:

* Files named `test_*.py` (or `*_test.py`), typically in a `tests/` directory.
* Inside those files, functions named `test_*`.

Running the `pytest` command from the repository root collects and runs everything that matches.

### Writing a test: `assert` is all you need

A test is a plain function that uses Python's built-in `assert` statement. If every assert holds, the test
passes; if one fails, pytest reports it with a detailed explanation of the values involved. 

Let's look at the tests for `histograms.py`. They are in `tests/test_histograms.py`.

A basic check: every histogram has at least one bin, and the lower edge is below the upper edge.
```python
import histograms

def test_ranges_are_valid():
    for variable, (nbins, low, high) in histograms.ranges.items():
        assert nbins > 0, f"{variable} has no bins"
        assert low < high, f"{variable} has an empty or inverted range"
```

`bookHistogram` books a histogram of a variable from an `RDataFrame`, so let's build a tiny dataset in memory 
(no input file needed!) and check the result:

```python
import ROOT

def test_book_histogram():
    # 100 events with pt_1 = 30 GeV and unit weight
    df = ROOT.RDataFrame(100).Define("pt_1", "30.0").Define("weight", "1.0")
    h = histograms.bookHistogram(df, "pt_1", histograms.ranges["pt_1"])
    # Number of bins must be the same
    assert h.GetNbinsX() == histograms.ranges["pt_1"][0]
    # Entries must be the same as the number of events in the dataset
    assert h.GetEntries() == 100
```

`test_histograms.py` also contains examples about Fixtures and Parametrization, that you can explore in the file itself.

These are examples of **unit tests**: they check individual functions of `histograms.py` with small in-memory datasets, 
so they are fast enough to run on every push.

### Putting it together

To run the tests at `tests/test_histograms.py`, you need to have `pytest` installed, and run the following 
commands from the repository root:

```bash
pytest                                                # run everything
pytest -v                                             # verbose: one line per test
pytest tests/test_histograms.py                       # run one file
pytest tests/test_histograms.py::test_book_histogram  # run one test
```

Note: ROOT is required to run the tests. If using the JupyterHub instance provided for this tutorial, 
you can run the tests in a terminal with the following commands:
```
apptainer exec docker://rootproject/root:6.32.04-ubuntu24.04 pytest
```

## Running tests in GitHub Actions

You probably already guessed it: you can add a job on the GitHub Actions pipeline to run the tests.

::::{admonition} Add a test job
:class: important
Add a `test` job to `.github/workflows/main.yml` that installs pytest and runs the test suite.

Think about two things:

1. Which container image does the job need?
2. Does it need to wait for any of the other jobs (`needs`), or can it run in parallel with them?

:::{admonition} Solution
:class: dropdown
```yaml
  test:
    runs-on: ubuntu-latest
    container: rootproject/root:6.32.04-ubuntu24.04
    steps:
      - name: checkout repository
        uses: actions/checkout@v6

      - name: install pytest
        run: apt-get update && apt-get install -y python3-pytest

      - name: run tests
        run: pytest -v
```

Note that we install pytest with `apt-get` (as we did for the XRootD client) rather than `pip install pytest`:
the `rootproject/root` image does not ship `pip`, and on Ubuntu 24.04 the system Python refuses `pip install`
outside a virtual environment anyway.

The job needs the ROOT container, since the tests import ROOT. But it has no `needs`: our tests build
their small datasets in memory, so they don't depend on the output of `build_skim`, `skim` or `plot` —
GitHub Actions runs the `test` job in parallel with the rest of the pipeline, and you get the test
verdict without waiting for the skimming to finish.
:::
::::

If any assert fails, pytest exits with a non-zero status code, the step fails, and the whole workflow is
marked as failed — exactly the red ❌ next to the commit that tells you (and your collaborators) not to
trust that version of the code. 

Try it out: break the `ranges` on purpose (e.g. unvalid range in a variable), push, and watch the parametrized tests catch it.

When a test fails, it should be easy to detect what was expected and determine how
to pinpoint the problem. For this, it is important to report the test results in a way that is easy to understand.
As the number of tests implemented scale up, it is also important to quickly identify the tests that are failing, and 
the main reason for the failure. Looking at the pipeline log is not the best way to do this.

GitLab CI provides a way to report test results in a standard format, so that they can be easily visualized in the
pipeline results. This is done by using the [JUnit](https://junit.org/junit5/) test report XML format (JUnit is a popular
testing framework for Java, but the XML format is language-agnostic and the reports have become a standard).

Each framework has its own way to generate the JUnit XML report (check the documentation of your favorite tool). 
For CTest, we can use the `--output-junit <file>` option. Once the XML is generated, we need to upload it as an artifact
The `.gitlab-ci.yml` file looks then like this:

```yaml
stages:
  - build
  - run
  - test
    
before_script:
  - mkdir -p cpp/analysis/build

build_code:
  stage: build
  image: rootproject/root:6.26.10-ubuntu22.04
  script:
    - cd cpp/analysis/build
    - cmake ../
    - make
  artifacts:
    paths:
      - cpp/analysis/build/
build_code_latest:
  stage: build
  image: rootproject/root:latest
  script:
    - cd cpp/analysis/build
    - cmake ../
    - make
  allow_failure: true
  
make_histograms:
  stage: run
  image: rootproject/root:6.26.10-ubuntu22.04
  dependencies:
    - build_code
  script:
    - cd cpp/analysis/build
    - time ./src/double_muon_analysis ../data/DoubleMu.root
  artifacts:
    paths:
      - cpp/analysis/build/histograms.root
    expire_in: 1 day

test_histograms:
  stage: test
  image: rootproject/root:6.26.10-ubuntu22.04
  dependencies:
    - build_code
    - make_histograms
  script:
    - cd cpp/analysis/build
    - ctest --output-on-failure
```

Let's commit and push the changes, and check the pipeline results.


## Report test results

When a test fails, it should be easy to detect what was expected and determine how
to pinpoint the problem. For this, it is important to report the test results in a way that is easy to understand.
As the number of tests implemented scale up, it is also important to quickly identify the tests that are failing, and 
the main reason for the failure. Looking at the pipeline log is not the best way to do this.

Actions provide a way to report test results in a standard format, so that they can be easily visualized in the
pipeline results. This is done by using the [JUnit](https://junit.org/junit5/) test report XML format (JUnit is a popular
testing framework for Java, but the XML format is language-agnostic and the reports have become a standard).

Each framework has its own way to generate the JUnit XML report (check the documentation of your favorite tool). 
For Pytest, we can use the `--junitxml= <file>` option. Once the XML is generated, we need to tell GitHub where to find it. 
This is done by the action `test-summary/action@v1`:

```yaml
    - name: Create test summary
      uses: test-summary/action@v1
      with:
        paths: <file>
      if: always()
```

It is important to add the `when: always` option, so that the report is generated even if the test fails.

Adding the report, our YAML file now looks like this:

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

  test:
    runs-on: ubuntu-latest
    container: rootproject/root:6.32.04-ubuntu24.04
    steps:
      - name: checkout repository
        uses: actions/checkout@v6

      - name: install pytest
        run: apt-get update && apt-get install -y python3-pytest

      - name: run tests
        run: pytest -v --junitxml=pytest.xml

      - name: Create test summary
        uses: test-summary/action@v1
        with:
          paths: pytest.xml
        if: always()
```

Update the pipeline, and check the results. Can you figure out where the test report is on the Web interface?
If something fails, how to quickly identify the source of the failure?


## Extend your pipeline

We have covered the basics, and now you have a working CI/CD pipeline that builds your code, runs it, and runs a test.
You can extend it to add more stages, more tests, and more features.

There is a lot more to learn about, and from now on is your turn to explore and learn more about it.
Keep at hand the [GitHub Actions reference](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax), and 
see how you can extend your pipeline to cover the needs of your projects.

Whenever you start a new project, keep in mind the tests you want to run, and the stages you want to have in your pipeline. 
You can use this repository as a template, and you will have a working pipeline ready to use. 


:::{admonition} Key Points
:class: note
- pytest discovers tests by naming convention: `test_*` functions in `test_*.py` files, and a plain `assert` is all a test needs.
- A failing test fails the CI job, flagging the commit before broken code reaches your collaborators.
:::
