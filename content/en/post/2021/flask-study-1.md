---
layout: post
title: Python Flask 1 Project Initialization
category: flask
date: 2021-09-05
tags:
- Python
- Flask 
- REST API
- Swagger UI
- docker
---

# Python Flask Best Practices: 1) Project Initialization

## Preface

Python Flask is a simple and convenient web framework.

It’s easy to build a web site or a pure Web API with it.

I recently needed a small web + scripting project. While setting up Flask, I couldn’t find a starter that fit my taste, so I explored a bit, hit a few bumps, and then put together this tutorial on the shoulders of giants.

## Project Structure

For a real project, I recommend a layered architecture (3-tier) + MVC separation to keep the code well organized. If you aren’t familiar with these concepts, it’s worth a quick read.

Notes:

- `src`: all project source files
- `src/service`: service/business logic
- `src/model`: business entities
- `src/db`: database related, including model definitions and DAO/SQL
- `src/sdk`: external integrations/SDKs
- `src/job`: background jobs (often triggered via API)
- `src/utils`: utilities; configs live here too
- `src/app.py`: Flask app entrypoint
- `manage.py`: flask.cli entry, starts API + jobs
- `Dockerfile`: Docker build
- `debug.py`: local debug entry
- `requirements.txt`: all dependencies
- `.vscode/launch.json`: VS Code debug config

## Let’s Build

Personal preference: I like managing dependencies via a `requirements.txt`. Use your preferred approach if you like.

### requirements.txt

```txt
flask
flask-swagger
flask-swagger-ui
flask-bootstrap
SQLAlchemy
pymysql
pydantic
requests
loguru
gunicorn
```

Brief notes:

- `flask` is the core; `flask-swagger` + `flask-swagger-ui` provide Swagger UI; `flask-bootstrap` helps quickly scaffold HTML pages.
- `SQLAlchemy` + `pymysql`: ORM + MySQL driver.
- `pydantic`: works with SQLAlchemy for model conversion, easing serialization/deserialization quirks.
- `requests`: call external HTTP APIs or write crawlers.
- `loguru`: simple logging; `from loguru import logger` and you’re set.
- `gunicorn`: multi-process deployment.

