---
title: Migrate from Pip requirements.txt to Poetry
tags:
  - python
  - poetry
  - pip
categories: Programming
date: 2024-01-05 18:04:24
---


Dependency management is a critical part of project development. If it were done wrong, project would behave differently between development and production environments. With Python, we have the tool `virtualenv` that isolates the project's environment from the system's, and we use `pip` and a `requirement.txt` file to maintain the list of dependencies. For instance:

```
Flask==3.0.0
Flask-SQLAlchemy==3.1.1
-e .
```

And the environment can be setup by:

```
% python3 -m venv venv
% source venv/bin/activate
(venv) % pip install -r requirements.txt
```


## Disadvantages of the `requirements.txt` approach

There are several shortcomings of this method. First and the major problem is this file only contains the *direct* dependencies, not the transitive ones. `pip freeze` shows that Flask and Flask-SQLAlchemy depend on several other packages:

```
% pip freeze
Flask==3.0.0
Werkzeug==3.0.1
Jinja2==3.1.2
Flask-SQLAlchemy==3.1.1
SQLAlchemy==2.0.25
-e git+ssh://git@github.com/jizhang/blog-demo@82e4b4c4c6e72ed44e0cce9ee45aca5abc4dc87b#egg=poetry_demo&subdirectory=poetry-demo
...
```

Take Werkzeug for an example. It is required by Flask, and its version is stated as `Werkzeug>=3.0.0` in Flask's [project file][1]. This may cause a problem when Werkzeug bumps its version to 4.x and after a reinstallation of the project, it will download the latest version of Werkzeug and create a compatibility issue. Same thing may happen to Flask-SQLAlchemy since functions of SQLAlchemy may vary between major versions.

A possible solution is to freeze the dependencies altogether:

```
% pip freeze >requirements.txt
```

This is practically the *lockfile* we see in other languages like `yarn.lock` and `Gemfile.lock`, whereby the installation process is fully reproducible. But for Python, we need extra effort to ensure that the `requirements.txt` is updated correctly after modifying the dependencies. And it also makes it difficult to upgrade the direct dependencies because the transitive ones need to be upgraded manually.

<!-- more -->

Another problem of `requirements.txt` is when we need to maintain different dependencies across environments. For instance, in development we may want to include `ruff` and `mypy`, and in production, `gunicorn` is required. In practice, we create two separate files for this purpose, the `requirements.txt`:

```
Flask==3.0.0
gunicorn==21.2.0
-e .
```

And `requirements-dev.txt`:

```
ruff==0.1.11
mypy==1.8.0
```

Then add a `Makefile` to facilitate the installation:

```
install:
        venv/bin/pip install -r requirements.txt

dev:
        venv/bin/pip install -r requirements.txt -r requirements-dev.txt
```


## Lock dependency versions with `poetry.lock`

Though `requirements.txt` has some problems, we have been using this approach in production for years. And thanks to the opensource community, we now have a better option, [Poetry][2]. I would like to call it [Yarn][7] for Python, because I maintain several frontend projects and always hope there would a tool that is as effective and easy-to-use as Yarn. With Poetry, we can simply use `poetry install` to lock dependency versions. And it does much more than that.

![Poetry](/images/poetry.png)

There are several ways to install Poetry. One can follow the instructions on its [official documentation][3]. For Homebrew users, simply use `brew install poetry`. Later in this article I will discuss how to use Poetry when deploying application with Docker. Here are some frequently used commands:

* `poetry init` starts an interactive tool to initialize an existing project. When finished, it will generate a `pyproject.toml` file at the root of the project.
* `poetry install` creates a virtual environment, installs dependencies according to the specification in `pyproject.toml`, and writes the `poetry.lock` file. Make sure you commit this file to the repository, so that other people can install the exact same versions when they clone your repository and run `poetry install`.
* `poetry add flask` adds the latest version of Flask to project's dependencies. The preferable way to specify version is using the caret symbol with [Semantic Versioning][4]. For example `^3.0.0` means the version ranges from `3.0.0` (inclusive) to `4.0.0` (exclusive).
* `poetry lock` should be used when you have manually edited the `pyproject.toml` file, so that `poetry.lock` can be updated accordingly.
* `poetry run flask run` starts the Flask development server within the virtualenv.


## Where is the virtual environment?

Unlike the traditional approach, Poetry does not create a `venv` folder in project's root. Instead, it collects all virtual environments in one place, for different projects and different Python versions.

```
% ls $(poetry config virtualenvs.path)
poetry-demo-uxUFVcdN-py3.11
poetry-demo-uxUFVcdN-py3.12
timetable-i_YWx-to-py3.11
```

When you have `python3.11` and `python3.12` in the `PATH`, you can use `poetry env use 3.12` to switch versions. Poetry also works with [pyenv][5], or even [custom-built Python binary][6].

If you prefer creating the virtual environment under the project's directory, set `virtualenvs.in-project` to `true`. Configurations are by default set globally. Add `--local` to set it [locally in the project][9]. Another option is to create virtualenv on your own. Make sure the folder is named `.venv` and Poetry will automatically pick it up.

```
% poetry env remove --all
Deleted virtualenv: poetry-demo-uxUFVcdN-py3.11
Deleted virtualenv: poetry-demo-uxUFVcdN-py3.12
% poetry config virtualenvs.in-project true
% poetry install
Creating virtualenv poetry-demo in /poetry-demo/.venv
```

We can also install dependencies into the system's environment. We will look into that in the Docker section.


## Manage dependencies for different environments

As mentioned above, we need different dependencies for development and production environment. Specifically, we want `Flask`, `ruff`, `mypy` to be installed in development environment, and `Flask`, `gunicorn` to be installed in production. To achieve that, we put `ruff` and `mypy` in a dependency group named `dev`, and specify `gunicorn` as a package extra. Here's what we do in the `pyproject.toml`:

```toml
[tool.poetry.dependencies]
python = "^3.11"
flask = "^3.0.0"
gunicorn = {version = "^21.2.0", optional = true}

[tool.poetry.group.dev.dependencies]
ruff = "^0.1.11"
mypy = "^1.8.0"

[tool.poetry.extras]
gunicorn = ["gunicorn"]
```

In development environment, we simply use `poetry install`. In production, use the following command:

```
poetry install --without dev --extras gunicorn
```


## Deploy application with Poetry and Docker

```Dockerfile
FROM python:3.11-slim

ENV POETRY_HOME=/opt/poetry
RUN python3 -m venv $POETRY_HOME && \
    $POETRY_HOME/bin/pip install poetry==1.7.1 && \
    $POETRY_HOME/bin/poetry config virtualenvs.create false

WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN $POETRY_HOME/bin/poetry install --without dev --extras gunicorn

COPY poetrydemo/ ./poetrydemo/
```

* It is advised to [install Poetry in a dedicated virtual environment][3], so we create one and install Poetry via `pip`.
* Since the container only has one application, it is safe to install project's dependencies into the system environment. Simply set `virtualenvs.create` to `false`. Or you can let Poetry create a dedicated one for the project.
* Project code resides in `/app`, and it is set as the working directory. So when `gunicorn` is called, which is available at system level, it can find `poetrydemo` package without problem.

```
% docker build -t poetrydemo .
% docker run -p 8000:8000 poetrydemo gunicorn -b 0.0.0.0:8000 poetrydemo:app
```


## Use a PyPI mirror

Last but not least, if you are in an area where internet access is restricted, the usual mirror config in `~/.pip/pip.conf` does not work, since [Poetry only processes its own config files][8]. To use a mirror for PyPI, add the following config into `pyproject.toml`:

```toml
[[tool.poetry.source]]
name = "aliyun"
url = "https://mirrors.aliyun.com/pypi/simple/"
priority = "default"
```


## References

* https://unbiased-coder.com/python-poetry-vs-pip/
* https://kennethreitz.org/essays/2016/02/25/a-better-pip-workflow
* https://github.com/pdm-project/pdm#comparisons-to-other-alternatives


[1]: https://github.com/pallets/flask/blob/735a4701d6d5e848241e7d7535db898efb62d400/pyproject.toml#L23
[2]: https://python-poetry.org/
[3]: https://python-poetry.org/docs/#installation
[4]: https://semver.org/
[5]: https://github.com/pyenv/pyenv
[6]: https://python-poetry.org/docs/managing-environments/#switching-between-environments
[7]: https://yarnpkg.com/
[8]: https://github.com/python-poetry/poetry/issues/1554#issuecomment-553113626
[9]: https://python-poetry.org/docs/configuration/#local-configuration
