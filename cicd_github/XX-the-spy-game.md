# Getting into the Spy Game

:::{admonition} Overview
:class: note
**Teaching:** 5 min | **Exercises:** 10 min

**Questions**
- How can I give my GitHub actions private information?

**Objectives**
- Add custom environment variables
- Learn how to give your CI/CD Runners access to private information
:::

:::{admonition} Recall
:class: important
```yaml
build_skim:
  needs: greeting
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
    - name: checkout repository
      uses: actions/checkout@v6

    - uses: actions/download-artifact@v8
      with:
        name: skim6.32.04-ubuntu24.04

    - name: skim
      run: |
        chmod +x ./skim
        ./skim
```
:::

In the previous lesson, we saw that the executable `skim` takes 5 arguments: input (remote data), output (processed data), cross-section, integrated luminosity, and scale.

Let's consider the following values
```
input: root://eosuser.cern.ch//eos/user/g/gstark/AwesomeWorkshopFeb2020/GluGluToHToTauTau.root
output: skim_ggH.root
cross_section: 19.6
integrated_luminosity: 11467.0
scale: 0.1
```

The input file sits in a **personal, access-protected EOS area** (here, a past instructor's). If you're a CERN user, you would use a path in your own area, `/eos/user/<initial>/<username>/…` — the point is that any protected path fails in the same way without authentication.

Our YAML file should look like
```yaml
...
skim:
  needs: build_skim
  runs-on: ubuntu-latest
  container: rootproject/root:6.32.04-ubuntu24.04
  steps:
    - name: checkout repository
      uses: actions/checkout@v6

    - uses: actions/download-artifact@v8
      with:
        name: skim6.32.04-ubuntu24.04

    - name: skim
      run: |
        chmod +x ./skim
        ./skim root://eosuser.cern.ch//eos/user/g/gstark/AwesomeWorkshopFeb2020/GluGluToHToTauTau.root skim_ggH.root 19.6 11467.0 0.1
```

What about the output?
```text
>>> Process input: root://eosuser.cern.ch//eos/user/g/gstark/AwesomeWorkshopFeb2020/GluGluToHToTauTau.root
Error: n <TNetXNGFile::Open>: [ERROR] Server responded with an error: [3010] Unable to give access - user access restricted - unauthorized identity used ; Permission denied
```

## Access Control

The data we're using are on CERN User Storage (EOS). As a general rule, access to protected data must be authenticated — you can't just grab it!
It means we need to give our GitHub Actions access to our data. CERN uses Kerberos (`kinit`) for access control.

Anyhow, this is pretty much done by executing `echo "$USER_PASS" | kinit "$USER_NAME@CERN.CH"` assuming that we've set the corresponding environment variables. One practical detail: unlike the old conda-based ROOT images, the Ubuntu-based `rootproject/root` images don't ship a Kerberos client, so the job has to install it first with `apt-get update && apt-get install -y krb5-user` (you'll see this in the solution below).

If you are not a CERN user, don't worry. We have a backup solution for you!
You can use this file `root://eospublic.cern.ch//eos/root-eos/HiggsTauTauReduced/GluGluToHToTauTau.root` and skip the rest of this lesson.

:::{admonition} Passwords in CI are not the real-world approach
:class: caution
CERN accounts use two-factor authentication these days, so piping your personal password into `kinit` from CI will generally not work for your own account — and storing a personal password in CI is bad practice anyway. The real-world pattern is a dedicated **service account** with restricted permissions (or short-lived tokens/keytabs). We keep the `kinit` example because the *mechanics* — storing a credential as a secret and using it in a job — are exactly the same.
:::

:::{admonition} Running example
:class: tip
Sometimes you'll run into a code example here that you might want to run locally but relies on variables you might not have set? Sure, simply do the following
```bash
USER_NAME=GoodWill
USER_PASS=hunter42
echo "$USER_PASS" | kinit "$USER_NAME@CERN.CH"
```
:::

## GitHub secrets

We first have to store our sensitive information in GitHub:

1. Navigate to the main page of the repository.
2. Select `Settings`.
3. In the left sidebar, go to `Secrets and variables`, then `Actions`, and then `New repository secret`.
4. Type `USER_NAME` in the Name input box and add your username in the Secret input box.
5. Similarly add `USER_PASS` as well.

:::{admonition} DON'T PEEK
:class: warning
DON'T PEEK AT YOUR FRIEND'S SCREEN WHILE DOING THIS.
:::

### Naming your secrets

Note that there are some rules applied to secret names:

- Secret names can only contain alphanumeric characters ([a-z], [A-Z], [0-9]) or underscores (_). Spaces are not allowed.
- Secret names must not start with the GITHUB_ prefix.
- Secret names must not start with a number.
- Secret names must be unique at the level they are created at. For example, a secret created at the organization-level must have a unique name at that level, and a secret created at the repository-level must have a unique name in that repository. If an organization-level secret has the same name as a repository-level secret, then the repository-level secret takes precedence.

:::{admonition} Access secrets
:class: important
The secrets you've created are available to use in GitHub Actions workflows. GitHub allows to access them using the secrets context: `${{ secrets.<secret name> }}`.

The recommended way to use them is to map each secret to an environment variable with `env:`, and only reference the variables in your shell commands:

```yaml
- name: access control
  env:
    USER_NAME: ${{ secrets.USER_NAME }}
    USER_PASS: ${{ secrets.USER_PASS }}
  run: echo "$USER_PASS" | kinit "$USER_NAME@CERN.CH"
```

Prefer this over interpolating `${{ secrets.X }}` directly inside `run:` — direct interpolation splices the secret into the shell script text itself, which is easier to leak (e.g. through quoting mistakes) than an environment variable.
:::

:::{admonition} Further Reading
:class: seealso
- [https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets)
:::

## Adding Artifacts on Success

As it seems like we have a complete CI/CD that does physics - we should see what came out. We just need to add artifacts for the `skim` job. This is left as an exercise to you.

::::{admonition} Adding Artifacts
:class: important
Let's add `artifacts` to our `skim` job to save the `skim_ggH.root` file. Let's have the artifacts expire in a week instead.

:::{admonition} Solution
:class: dropdown
```yaml
...
skim:
  needs: build_skim
  runs-on: ubuntu-latest
  container: rootproject/root:6.32.04-ubuntu24.04
  steps:
    - name: checkout repository
      uses: actions/checkout@v6

    - uses: actions/download-artifact@v8
      with:
        name: skim6.32.04-ubuntu24.04

    - name: access control
      env:
        USER_NAME: ${{ secrets.USER_NAME }}
        USER_PASS: ${{ secrets.USER_PASS }}
      run: |
        apt-get update && apt-get install -y krb5-user
        echo "$USER_PASS" | kinit "$USER_NAME@CERN.CH"

    - name: skim
      run: |
        chmod +x ./skim
        ./skim root://eosuser.cern.ch//eos/user/g/gstark/AwesomeWorkshopFeb2020/GluGluToHToTauTau.root skim_ggH.root 19.6 11467.0 0.1

    - uses: actions/upload-artifact@v7
      with:
        name: skim_ggH
        path: skim_ggH.root
        retention-days: 7
```
:::
::::

And this allows us to download artifacts from the successfully run job.

:::{admonition} Key Points
:class: note
- Secrets in GitHub actions allow you to hide protected information from others who can see your code
:::
