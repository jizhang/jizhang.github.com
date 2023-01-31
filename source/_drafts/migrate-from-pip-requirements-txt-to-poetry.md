---
title: Migrate from Pip requirements.txt to Poetry
tags: [python, poetry, pip]
categories: Programming
---

TODO

<!-- more -->

* ~~pypi mirror~~
    * default repository
* ~~different python version~~
    * pyenv local 3.10
    * poetry env use 3.10
* ~~where is venv~~
    * cache dir, multiple venv
    * poetry config virtualenvs.in-project true --local, .venv
    * vscode
* build docker
* ~~requirements.txt~~
    * remove setup.py
    * ^3.10, ~3.10
    * -e . by default
    * poetry add --group dev pylint
    * --extras gunicorn
* ~~Makefile~~
* pre-commit

Different ways to use poetry with docker
* export requirements.txt and use pip in docker
* build .whl
* use poetry inside Dockerfile, multistage build
    * pipx
    * multi-stage build
    * .venv vs system
* docker/build-push-action, cache
* mimic kiwi

## References
* https://unbiased-coder.com/python-poetry-vs-pip/
