---
title: Setup CI with GitHub Actions (Java/Node/Python)
tags:
  - github
  - ci
  - docker
  - java
categories: Programming
date: 2023-02-13 18:36:12
---


Continuous integration, or CI, is a great tool to maintain a healthy code base. As in [lint-staged][1]'s motto, "don't let 💩 slip into your code base", CI can run various checks to prevent compilation error, unit test failure, or violation of code style from being merged into the main branch. Besides, CI can also do the packaging work, making artifacts that are ready to be deployed to production. In this article, I'll demonstrate how to use [GitHub Actions][2] to define CI workflow that checks and packages Java/Node/Python applications.

![CI with GitHub Actions](images/ci-with-github-actions.png)

## Run Maven verify on push

CI typically has two phases, one is during development and before merging into the master, the other is right after the feature branch is merged. Former only requires checking the code, i.e. build the newly pushed code in a branch, and see if there's any violation or bug. After it's merged, CI will run checking *and* packaging altogether, to produce a deployable artifact, usually a Docker image.

For Java project, we use JUnit, Checkstyle and SpotBugs as Maven plugins to run various checks whenever someone pushes to a feature branch. To do that with GitHub Actions, we need to create a workflow that includes setting up Java environment and running `mvn verify`. Here's a minimum workflow definition in `project-root/.github/workflows/build.yml`:

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
* [actions/checkout][4] merely checks out the code into workspace for further use. It only checks out the one commit that triggers this workflow. It's also a good practice to pin the version of an action, as in `actions/checkout@v3`.
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

Before running the `verify` job, the runner, with Docker already installed, starts up a Redis container and maps its port to the host, in this case `6379`. Then any process in the runner can access Redis via `localhost:6379`. Mind that containers take time to start, and sometimes the starting process is long, so GitHub Actions uses `docker inspect` to ensure container has entered the `healthy` state before it makes headway to the next step. So we need to set [`--health-cmd`][9] for our services:

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

After `mvn verify`, there'll be an uber JAR in `target/project-1.0-SNAPSHOT.jar`, and we want to build it into a Docker image for deployment. We're going to create a separate job for this task, but the first thing we need to do is to transfer the JAR file from the `verify` job *to* the new `build` job, because jobs in a workflow are executed independently and in parallel, so we also need to tell the runner that `build` is dependent on `verify`.

```yaml
env:
  JAR_FILE: project-1.0-SNAPSHOT.jar

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - run: mvn --batch-mode verify
      - uses: actions/upload-artifact@v3
        with:
          name: jar
          path: target/${{ env.JAR_FILE }}
          retention-days: 5

  build:
    needs: verify
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: jar
```

* `env` is a place to set common variables within workflow. Here we use it for the filename of the JAR. We'll see more use of it in the `build` job.
* `actions/upload-artifact` and its counterpart `download-artifact` are used to share files between jobs, aka., artifact. It can be a single file or a directory, identified by the `name`. Artifacts can only be shared within the same *workflow run*. Once uploaded, they are accessible through GitHub UI as well. There're more examples in the [documentation][11].
* `needs` creates a dependency between `verify` and `build`, so that they are executed sequentially.

## Build Docker image for deployment

Let's take Spring Boot project for an example. There're some [guidelines][12] on how to efficiently build the packaged JAR into a layered Docker image, with the built-in tool provided by Spring Boot. The full Dockerfile can be found in the above link. One thing we care about is the `JAR_FILE` argument:

```Dockerfile
FROM eclipse-temurin:17-jre as builder
WORKDIR application
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} application.jar
# ...
```

For the `build` job, the `docker` CLI is already installed in the runner, but we still need to take care of something like logging into Docker repository, tagging the image, etc. Fortunately there're some `actions` for these purposes. Besides, we're not going to push our image into Docker hub. Instead, we use the [GitHub Packages][13] service. Here's the full definition of the `build` job:

```yaml
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  JAR_FILE: project-1.0-SNAPSHOT.jar

jobs:
  build:
    if: github.ref == 'refs/heads/master'
    needs: verify
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: jar
      - uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/metadata-action@v4
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: type=sha
      - uses: docker/build-push-action@v3
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          build-args: |
            JAR_FILE=${{ env.JAR_FILE }}
```

* `if` statement indicates this job is only executed under certain circumstances. In this case, only run on `master` branch. There're other [conditions][14] you can use, and `if` can also be added in `step`. Say only upload artifact when the `verify` job is executed on `master` branch.
* `docker/login-action` setups the credentials for logging into GitHub Packages. The `GITHUB_TOKEN` is automatically generated and its permissions can be controlled in the [Settings][15].
* `docker/metadata-action` is used to extract meta data from the repository. In this example, I'm using the Git short commit as the Docker image tag, i.e. `sha-729b875`, and this action helps me to extract this information from the Git repository and exposes it as the [output][16], which is another feature that GitHub Actions provides for sharing information between steps and jobs. To be more specific:
    * `metadata-action` will generate a list of tags based on the `images` and `tags` parameters. The above configuration will generate something like `ghcr.io/jizhang/proton:sha-729b875`. Other options can be found in this [link][17].
    * We give this step an `id`, which is `meta`, and then access its output via `steps.meta.outputs.tags`.
    * The parameter `images`, `tags`, and `tags` in `build-push-action` all support multi-line string so that multiple tags can be published.
* `docker/build-push-action` does the build-and-push job. The Dockerfile should be in the project root, and we pass the `JAR_FILE` argument which points to the artifact that we've downloaded from the previous job.

If built successfully, the Docker image can be found in your Profile - Packages. Here's the [full example][18] of using GitHub Actions with a Java project. The final pipeline looks like this:

![GitHub Actions with Java project](images/github-actions-java.png)

## Setup CI for Node.js project

Similarly, we create two jobs for testing and building. In the `test` job, we use the official `setup-node` action to install specific Node.js version. It also privodes cache facility for `yarn`, `npm` package managers.

```yaml
jobs:
  test:
    steps:
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: yarn
      - run: yarn install --frozen-lockfile
      - run: yarn lint:ci

  build:
    if: github.ref == 'refs/heads/master'
    needs: test
    steps:
      - run: yarn build
      - uses: docker/build-push-action@v3
        with:
          context: .
          file: build/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
```

The build output is generaly in the `dist` directory, so we just copy it onto an Nginx image and publish to GitHub Packages. I also have a [project][19] for demonstration.

```Dockerfile
FROM nginx:1.17-alpine
COPY build/nginx/default.conf /etc/nginx/conf.d/default.conf
COPY dist/ /app/
```

## Setup CI for Python project

For Python project, one of the popular dependency management tools is [Poetry][20], and the official `setup-python` action provides out-of-the-box caching for Poetry-managed virtual environment. Here's the abridged `build.yml`, full file can be found in this [link][21].

```yaml
env:
  PYTHON_VERSION: '3.10'
  POETRY_VERSION: '1.3.2'

jobs:
  test:
    steps:
      - run: pipx install poetry==${{ env.POETRY_VERSION }}
      - uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: poetry
      - run: poetry install
      - run: poetry run ruff timetable
      - run: poetry run mypy timetable

  build:
    if: github.ref == 'refs/heads/master'
    needs: test
    steps:
      - uses: docker/build-push-action@v3
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          build-args: |
            PYTHON_VERSION=${{ env.PYTHON_VERSION }}
            POETRY_VERSION=${{ env.POETRY_VERSION }}
```

But for the `build` job, Python is different from the aforementioned projects in that it doesn't produce bundle files like JAR or minified JS. So we have to invoke Poetry inside the Dockerfile to install the project dependencies, which makes the Dockerfile a bit more complicated:

```Dockerfile
ARG PYTHON_VERSION
FROM python:${PYTHON_VERSION}-slim

ARG POETRY_VERSION
ENV POETRY_HOME=/opt/poetry
RUN python3 -m venv $POETRY_HOME && \
    $POETRY_HOME/bin/pip install poetry==${POETRY_VERSION} && \
    $POETRY_HOME/bin/poetry config virtualenvs.create false

WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN $POETRY_HOME/bin/poetry install --extras gunicorn --without dev

COPY timetable/ ./timetable/
```

* According to the guidelines, Poetry should be installed in a separate virtual environment. Using `pipx` also works.
* For project dependencies however, we install them directly into the system level Python, because this container is only used by one application. Setting `virtualenvs.create` to `false` tells Poetry to skip creating new environment for us.
* When installing dependencies, we skip the ones for development and include the `gunicorn` WSGI server. Check out the documentation of Poetry and the sample project's [pyproject.toml][22] file for more information.

## References

* https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-java-with-maven
* https://docs.github.com/en/actions/publishing-packages/publishing-docker-images
* https://endjin.com/blog/2022/09/continuous-integration-with-github-actions
* https://github.com/vuejs/vue/blob/v2.7.14/.github/workflows/ci.yml


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
[11]: https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts
[12]: https://docs.spring.io/spring-boot/docs/current/reference/html/container-images.html
[13]: https://docs.github.com/en/packages
[14]: https://docs.github.com/en/actions/learn-github-actions/contexts
[15]: https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#setting-the-permissions-of-the-github_token-for-your-repository
[16]: https://docs.github.com/en/actions/using-jobs/defining-outputs-for-jobs
[17]: https://github.com/docker/metadata-action
[18]: https://github.com/jizhang/proton-server/blob/32b5a28f5c7227d74557a1e80dc6579b345487a1/.github/workflows/build.yml
[19]: https://github.com/jizhang/proton/blob/2da93e759861236099983955ef4964958a70248d/.github/workflows/build.yml
[20]: https://python-poetry.org/
[21]: https://github.com/jizhang/timetable/blob/63a77df1a2f0df4d1e816e60211f1e960441029b/.github/workflows/build.yml
[22]: https://github.com/jizhang/timetable/blob/63a77df1a2f0df4d1e816e60211f1e960441029b/pyproject.toml
