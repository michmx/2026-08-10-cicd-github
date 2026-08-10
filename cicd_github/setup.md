# Setup

## Set up the examples repository

1. Create a fork of [https://github.com/michmx/2026-08-10-cicd-github-examples](https://github.com/michmx/2026-08-10-cicd-github-examples)

   Click [HERE](https://github.com/michmx/2026-08-10-cicd-github-examples/fork), or alternatively the "Fork" button in the upper right corner of the page to create a copy of the repository under your GitHub account.


2. Get the code

   Open a terminal and clone the repository that contains files required for this lesson.

  ```bash
  git clone git@github.com:<YOURUSERNAME>/2026-08-10-cicd-github-examples.git
  ```

  :::{admonition} HTTPS instead of SSH
  :class: tip
  The commands above use SSH URLs, which require an [SSH key set up with your GitHub account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh). If you authenticate over HTTPS instead (e.g. with the `gh` CLI), clone with

  ```bash
  gh auth login
  git clone https://github.com/<YOURUSERNAME>/2026-08-10-cicd-github-examples.git
  ```
  :::

Done!

If you're having issues, **please let us know immediately**
since you might not be able to follow this lesson without a proper setup.
