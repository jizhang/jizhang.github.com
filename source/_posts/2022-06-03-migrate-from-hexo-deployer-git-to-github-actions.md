---
title: Migrate from hexo-deployer-git to GitHub Actions
tags:
  - hexo
  - github
categories: Programming
date: 2022-06-03 14:34:18
---


## TL;DR

Create `.github/workflows/pages.yml` in your `master` branch:

```yaml
name: Update gh-pages

on:
  push:
    branches:
      - master

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: "12.22"
          cache: yarn
      - run: yarn install
      - run: yarn build
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

Go to GitHub repo's Settings > Pages, change source branch to `gh-pages`.

## How it works

<!-- more -->

Previously with [`hexo-deployer-git`][1] plugin, we generate the static site locally and push those files to github's master branch, which will be deployed to GitHub Pages server. The config in `_config.yml` is as simple as:

```yaml
deploy:
  type: git
  repo: git@github.com:jizhang/jizhang.github.com
```

Now with GitHub Actions, a CI/CD platform available to public repositories, the build process can be triggered on remote servers whenever master branch is updated. Hexo provides an [official document][2] on how to setup the workflow, but it turns out the configuration can be a little bit simpler, thanks to the new versions of `actions` (we'll cover it later).

A workflow is a sequence of jobs to build, test, and deploy our code. Here we only need one job named `deploy` to generate the static files and push to a branch.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps: []
```

A job consists of steps that either run a shell command or invoke an `action` to execute a common task. For instance, we have defined two steps to install node dependencies and build the static site:

```yaml
steps:
  - run: yarn install
  - run: yarn build
```

Make sure you have the following scripts in `package.json`. Newer version of hexo already has them.

```json
{
  "name": "hexo-site",
  "private": true,
  "scripts": {
    "start": "hexo server --draft",
    "build": "hexo generate"
  }
}
```

But where does the node environment come from? First, the job `runs-on` a specified platform, which is `ubuntu-latest` here, and `uses` the [`setup-node`][3] action to prepare the node environment, `yarn` command, as well as the cache facility.

```yaml
steps:
  - uses: actions/setup-node@v3
    with:
      node-version: "12.22"
      cache: yarn
```

Under the hood, it searches for a local cache of the specific node version, where github provides last three LTS versions, or it falls back to downloading from the official site. The `yarn` package manager is pre-bundled by github, or you need a separate step to install it.

When it comes to caching the downloaded packages, `setup-node` action utilizes [`actions/cache`][4]. It caches the global package data, i.e. `~/.cache/yarn/v6` folder, instead of `node_modules`, so that cache can be shared between different node versions. `setup-node` generates a cache key in the form of `node-cache-Linux-yarn-${hash(yarn.lock)}`. See more about caching on [GitHub Docs][5].

The static site is generated in `public` folder, and we need to push them into the `gh-pages` branch. There is an action [`peaceiris/actions-gh-pages`][6] that already covers this. It first clones the `gh-pages` branch into work directory, overwites it with the files in `public` folder, commits and pushes to the remote branch. The `GITHUB_TOKEN` is provided by GitHub Actions, with adequate permissions.

```yaml
steps:
  - uses: peaceiris/actions-gh-pages@v3
    with:
      github_token: ${{ secrets.GITHUB_TOKEN }}
      publish_dir: ./public
```

Last but not least, this workflow needs to be triggered on the `push` event of the `master` branch:

```yaml
on:
  push:
    branches:
      - master
```

Here is a screenshot of this workflow:

![Use GitHub Actions](/images/use-github-actions.png)


[1]: https://github.com/hexojs/hexo-deployer-git
[2]: https://hexo.io/docs/github-pages
[3]: https://github.com/actions/setup-node
[4]: https://github.com/actions/cache
[5]: https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows
[6]: https://github.com/peaceiris/actions-gh-pages
