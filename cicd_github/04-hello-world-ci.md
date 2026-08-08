# Hello CI World

:::{admonition} Overview
:class: note
**Teaching:** 5 min | **Exercises:** 10 min

**Questions**
- How do I run a simple GitHub Actions job?

**Objectives**
- Add CI/CD to a new project.
:::

## Adding CI/CD to a project

Let's start with a project that doesn't have any CI/CD configured. We'll use the
[2026-08-10-cicd-github-examples](https://github.com/michmx/2026-08-10-cicd-github-examples) repository for this example.
Create a [fork of the repository](https://github.com/michmx/2026-08-10-cicd-github-examples/fork) and clone it to your local session:
```bash
GITHUB_USER=<your-github-username>
git clone https://github.com/$GITHUB_USER/2026-08-10-cicd-github-examples.git
```

:::{admonition} Permissions for remote repository
:class: tip
Be sure to have the credentials to write in the remote repository. 
If you don't, you won't be able to push your changes and trigger the CI/CD job.

Authenticate to GitHub from the command line, either with SSH keys or with the `gh` command line interface (see [caching your GitHub credentials in git](https://docs.github.com/en/get-started/git-basics/caching-your-github-credentials-in-git))
:::


The first thing we'll do is create a `.github/workflows/main.yml` file in the project.
```bash
cd 2026-08-10-cicd-github-examples/
mkdir -p .github/workflows
touch .github/workflows/main.yml
```

Open `.github/workflows/main.yml` with your favorite editor and add the following
```yaml
name: example
on: push
jobs:
  greeting:
    runs-on: ubuntu-latest
    steps:
      - run: echo hello world
```

:::{admonition} Working on Jupyter Hub
:class: tip
If you are following the tutorial on the Jupyter Hub instance, hidden files are not displayed by default.
Therefore, you can't see the `.github` directory and its contents outside the terminal. 

A trick to use the Jupyter Hub file browser to edit the `.github/workflows/main.yml` file is to create a symbolic link 
to it in your home directory. You can do this with the following command:
```bash
ln -s $PWD/.github/workflows $PWD/workflows
```
However, you can also edit the file directly from the terminal using your favorite text editor (e.g. `nano`, `vim`, `emacs`, etc.)
:::

## Run GitHub Actions

Now, let's see it in action!

We've created the `.github/workflows/main.yml` file, but it's not yet on GitHub. We'll push these changes to GitHub so that it can run our job.

Since we're adding a new feature (Actions) to our project, we'll work in a feature branch. This is just a 
human-friendly named branch to indicate that it's adding a new feature.

```bash
git add .github/workflows/main.yml
git commit -m "my first actions"
git push origin main
```

And that's it! You've successfully run your CI/CD job, and you can view the output. You just have to navigate to the GitHub webpage 
for your repository and hit the Actions tab, where you will find details of your job (status, output,...).

From this page, click through until you can find the output for the successful job run .


:::{admonition} Key Points
:class: note
- Creating `.github/workflows/main.yml` is the first step to salvation.
- Pipelines are made of jobs with steps.
:::
