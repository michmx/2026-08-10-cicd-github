# CI/CD with GitHub Actions

GitHub is a distributed git platform used for code hosting and collaboration. It can also be used to automatically run the hosted code on GitHub's servers via GitHub Actions. Actions are workflow automation scripts. We'll learn how to develop workflows that make our code robust to errors, preserved, and reproducible.

The aim of this module is to:
- explore what it means to build a CI/CD workflow
- guide you through building a CI/CD workflow

:::{admonition} Prerequisites
:class: caution
This assumes that you'll have some basic background with your command line, for example:

1. How to execute custom shell scripts (if you are not familiar with the shell, click [here](https://swcarpentry.github.io/shell-novice/))
2. How to run python scripts (if you are not familiar with python, click [here](https://swcarpentry.github.io/python-novice-inflammation/))
3. How to interact with remotes in git (if you are not familiar with git, click [here](https://swcarpentry.github.io/git-novice/), in particular the [Remotes in GitHub](https://swcarpentry.github.io/git-novice/07-github.html) episode)
4. How to authenticate to GitHub from the command line, either with SSH keys or with the `gh` command line interface (see [caching your GitHub credentials in git](https://docs.github.com/en/get-started/git-basics/caching-your-github-credentials-in-git))
:::

:::{admonition} The HSF Training CI/CD Lessons
:class: seealso
This is a condensed version of the HSF Training CI/CD with GitHub for [CompHEP 2026](https://indico.cern.ch/event/1672591/). The original lesson can be found [here](https://hsf-training.github.io/hsf-training-cicd-github/).

There is also a version of this lesson for GitLab CI/CD, which can be found [here](https://hsf-training.github.io/hsf-training-cicd/).

:::

## Table of Contents

```{tableofcontents}
```
