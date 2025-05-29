# Dockerfile
# Uses multi-stage builds requiring Docker 17.05 or higher
# See https://docs.docker.com/develop/develop-images/multistage-build/

# Creating a python base with shared environment variables
# docker build --name poetry-project --file docker/Dockerfile . --target python-base
FROM python:3.12-slim as python-base
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=off \
    PIP_DISABLE_PIP_VERSION_CHECK=on \
    PIP_DEFAULT_TIMEOUT=100 \
    POETRY_VIRTUALENVS_IN_PROJECT=true \
    POETRY_NO_INTERACTION=1 \
    POETRY_HOME="/opt/poetry" \
    PYSETUP_PATH="/opt/pysetup" \
    VENV_PATH="/opt/pysetup/.venv"
# /opt/pysetup/.venv/bin/fastapi

ENV PATH="$POETRY_HOME/bin:$VENV_PATH/bin:$PATH"


# builder-base is used to build dependencies
FROM python-base as builder-base
RUN apt-get update \
    && apt-get install --no-install-recommends -y \
        curl \
        build-essential

# Install Poetry - respects $POETRY_VERSION & $POETRY_HOME
ENV POETRY_VERSION=2.1.1
RUN curl -sSL https://install.python-poetry.org | python3 -

# We copy our Python requirements here to cache them
# and install only runtime deps using poetry
WORKDIR $PYSETUP_PATH
COPY ./poetry.lock ./pyproject.toml ./
# install runtime dependencies, no code is copied yet, and no dev deps
RUN poetry install --only main --no-root

CMD ["tail", "-f", "/dev/null"]

# use docker-compose to mount local volume under /app
FROM python-base as development
ENV FASTAPI_ENV=development

# Copying poetry and venv into image
COPY --from=builder-base $POETRY_HOME $POETRY_HOME
COPY --from=builder-base $PYSETUP_PATH $PYSETUP_PATH
COPY --from=builder-base /opt/pysetup/.venv/bin/fastapi /opt/pysetup/.venv/bin/fastapi

# Copying in our entrypoint
COPY ./docker/docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh

# venv already has runtime deps installed we get a quicker install
WORKDIR $PYSETUP_PATH
RUN poetry install

WORKDIR /app
COPY . .

ENTRYPOINT /docker-entrypoint.sh $0 $@
CMD ["poetry", "run", "uvicorn", "app.main:app", "--host=0.0.0.0", "--port=8000", "--reload"]
EXPOSE 80


FROM python-base as production
ENV FASTAPI_ENV=production
COPY --from=builder-base /opt/pysetup/.venv/bin/fastapi /opt/pysetup/.venv/bin/fastapi
COPY --from=builder-base $VENV_PATH $VENV_PATH

COPY ./docker/docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh
# RUN poetry install --only root

COPY ./app /app
WORKDIR /app

ENTRYPOINT /docker-entrypoint.sh $0 $@
CMD ["uvicorn", "main:app", "--host=0.0.0.0", "--port=80"]
EXPOSE 80
