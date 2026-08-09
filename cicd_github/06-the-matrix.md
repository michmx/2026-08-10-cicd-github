# Matrix

:::{admonition} Overview
:class: note
**Teaching:** 5 min | **Exercises:** 10 min

**Questions**
- How can we make job templates?

**Objectives**
- Don't Repeat Yourself (DRY)
- Use a single job for multiple jobs
:::

## Matrix

From the previous lesson, we tried to build the code against two different ROOT images by adding an extra job:

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

:::{admonition} Building a matrix across different versions
:class: tip

We could do better using `matrix`. The latter allows us to test the code against a combination of versions in a single job.

```yaml
jobs:
  greeting:
    runs-on: ubuntu-latest
    steps:
      - run: echo hello world

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
Beware of one YAML pitfall when listing versions in a matrix: YAML parses unquoted numbers, and a number like `3.10` is read as the float `3.1` (the trailing zero is dropped). A matrix like `version: [3.12, 3.13, 3.14]` would therefore run on unintended versions. This bit many projects when Python 3.10 was released — their matrices silently became `3.1`. Avoid it by always quoting version numbers as strings: `version: ['3.12', '3.13', '3.14']`.

More details on matrix: [https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idstrategymatrix](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idstrategymatrix).
:::

We can push the changes to GitHub and see how it will look like.
```bash
git add .github/workflows/main.yml
git commit -m "add multi jobs"
git push -u origin feature/add-actions
```

While the jobs are running, let's imagine we don't want our CI/CD to crash if that happens. We have to add `continue-on-error: true` to a job
```yaml
runs-on: ubuntu-latest
continue-on-error: true
```
For the matrix case, GitHub Actions cancels all in-progress and queued jobs *in the matrix* if any matrix job fails (the rest of the workflow is unaffected). This can be prevented by using the `fail-fast: false` key:value.
```yaml
strategy:
  fail-fast: false
```

More details: [https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idcontinue-on-error](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idcontinue-on-error).

::::{admonition} Allow a specific matrix job to fail
:class: important
But what if we want to **only** allow the job with version set to `latest` to fail without failing the workflow run?

:::{admonition} Solution
:class: dropdown

```yaml
runs-on: ubuntu-latest
container: rootproject/root:${{ matrix.version }}
continue-on-error: ${{ matrix.allow_failure }}
strategy:
  matrix:
    version: [6.32.04-ubuntu24.04]
    allow_failure: [false]
    include:
      - version: latest
        allow_failure: true
```
:::
::::

Look how much cleaner you've made the code. You should now see that it's pretty easy to start adding more build jobs for other images in a relatively clean way, as you've now abstracted the actual building from the definitions.

:::{admonition} Key Points
:class: note
- Using `matrix` allows to test the code against a combination of versions.
:::
