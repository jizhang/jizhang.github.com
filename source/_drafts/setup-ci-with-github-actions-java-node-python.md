---
title: Setup CI with GitHub Actions (Java/Node/Python)
tags: [github, ci, java, spring boot, docker]
categories: Programming
---

Continuous integration, or CI, is a great tool to maintain a healthy code base. As in [lint-staged][1]'s motto, "don't let 💩 slip into your code base", CI can run various checks to prevent compilation error, unit test failure, or violation of code style from being merged into the main branch. Besides, CI can also do the packaging work, making artifacts that are ready to be deployed to production. In this article, I'll demonstrate how to use [GitHub Actions][2] to define CI workflow that checks and packages Java/Node/Python applications.

![CI with GitHub Actions](images/ci-with-github-actions.png)

## Run Maven verify on push

CI typically has two phases, one is during development and before merging into the master, the other is right after the feature branch is merged. Former only requires checking the code, say build the newly pushed code in a branch, and see if there's any violation or bug. After it's merged, CI will run checking *and* packaging altogether, to produce a deployable artifact, a Docker image for example.

For Java project, we use JUnit, Checkstyle and SpotBugs as Maven plugins to run various checks whenever someone pushes to a feature branch. To do that with GitHub Actions, we need to create a workflow that includes setting up Java environment and run `mvn verify`. Here's a minimum workflow definition in `project-root/.github/workflows/build.yml`:

```yaml
name: Build
on: push
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: temurin
          cache: maven
      - run: mvn --batch-mode verify
```

<!-- more -->

* `on: push` defines the [trigger][3] of the workflow. Whenever there's a new commit pushed to any branch, the workflow will run. You can limit the branches that trigger this workflow, or use some other events like `pull_request`.
* `verify` is the name of a job we define in this workflow. A workflow can have multiple jobs, we'll add another one named `build` very soon. Jobs are executed in parallel by default, that's why `jobs` is a mapping instead of a sequence. But we can add dependencies between jobs, as well as conditions that may prevent a job from running.
* A job consists of severl `steps`, here we've defined three. A step can either be a command, indicated by `run`; or use of a predefined set of code, named "action", indicated by `uses`. There're tons of official and third-party actions we can use to build up a workflow. We can also build our own actions to share in a corporation.
* [actions/checkout][4] merely checks out the code into workspace for further use. It only checks out the one commit that triggers this workflow. It's also a good practice to pin the version of an action.
* [actions/setup-java][5] creates the specific JDK environment for us. `cache: maven` is important here because it utilizes the [actions/cache][6] to upload Maven dependencies to GitHub's cache server, so that they don't need to be downloaded from the central repository again. The cache key is based on the content of `pom.xml`, and there're several rules of [cache sharing between branches][7].

## Initialize service containers for testing

During the test phase, we oftentimes need a local database service to run the unit tests, integration tests, etc., and GitHub Actions comes with a ready-made solution for this purpose, viz. [Containerized services][8]. Here is a minimum example of spinning up a Redis instance within a job:

```yaml
jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:6
        ports:
          - 6379:6379
```

Before running the `verify` job, the runner, with Docker already installed, starts up a Redis container and maps its port to the host, in this case `6379`. Then any process in the runner can access Redis via `localhost:6379`. Mind that containers take time to start, and sometimes the starting process is long, so GitHub Actions uses `docker inspect` to ensure container has entered the `healthy` state before it makes headway to the next steps. So we need to set [`--health-cmd`][9] for our services:

```yaml
redis:
  image: redis:6
  ports:
    - 6379:6379
  options: >-
    --health-cmd "redis-cli ping"
    --health-interval 10s
    --health-timeout 5s
    --health-retries 5
```

This is especially important for the MySQL service we are about to setup, because it usually takes more time to start up:

```yaml
jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:5.7
        ports:
          - 3306:3306
        env:
          MYSQL_DATABASE: project_test
          MYSQL_ALLOW_EMPTY_PASSWORD: yes
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
```

## Share artifacts between jobs

## Build Docker image for deployment

## Setup CI for Node.js project

## Setup CI for Python project

* ~~Java & node project~~.
* ~~Lint in feature branches~~.
    * ~~Cache~~
* Forbid pull request from being merged if lint doesn't pass.
* ~~Test with mysql & redis~~.
* ~~Build docker to GitHub Packages~~.
    * Cache.
    * Clean up old versions.
* ~~GitHub Actions billing~~.

## References
* https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-nodejs
* https://docs.github.com/en/actions/publishing-packages/publishing-docker-images
* https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
* https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions

* https://github.com/actions/starter-workflows/blob/main/ci/node.js.yml
* https://github.com/actions/setup-node

* https://github.com/vuejs/vue/blob/main/.github/workflows/ci.yml

* https://endjin.com/blog/2022/09/continuous-integration-with-github-actions


[1]: https://github.com/okonet/lint-staged
[2]: https://docs.github.com/en/actions
[3]: https://docs.github.com/en/actions/using-workflows/triggering-a-workflow
[4]: https://github.com/actions/checkout
[5]: https://github.com/actions/setup-java
[6]: https://github.com/actions/cache
[7]: https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows
[8]: https://docs.github.com/en/actions/using-containerized-services/about-service-containers
[9]: https://docs.docker.com/engine/reference/commandline/run/
[10]: https://hub.docker.com/_/mysql
