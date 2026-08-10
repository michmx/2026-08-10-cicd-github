# Let's Actually Make A Test (For Real)

:::{admonition} Overview
:class: note
**Teaching:** 5 min | **Exercises:** 20 min

**Questions**
- How does a realistic workflow look for a physics analysis?

**Objectives**
- Actually add a test on the output of running physics
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

It is installed with pip:

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

```bash

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
        run: pip install pytest

      - name: run tests
        run: pytest -v
```

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

:::{admonition} Key Points
:class: note
- pytest discovers tests by naming convention: `test_*` functions in `test_*.py` files, and a plain `assert` is all a test needs.
- A failing test fails the CI job, flagging the commit before broken code reaches your collaborators.
:::
