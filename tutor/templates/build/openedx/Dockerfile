FROM docker.io/ubuntu:16.04
MAINTAINER Overhang.io <contact@overhang.io>

############ common to lms & cms

# Install system requirements
RUN apt update && \
    # Global requirements
    apt install -y language-pack-en git python-virtualenv build-essential software-properties-common curl git-core libxml2-dev libxslt1-dev python-virtualenv libmysqlclient-dev python-apt python-dev libxmlsec1-dev libfreetype6-dev swig gcc g++ \
    # openedx requirements
    gettext gfortran graphviz graphviz-dev libffi-dev libfreetype6-dev libgeos-dev libjpeg8-dev liblapack-dev libpng12-dev libsqlite3-dev libxml2-dev libxmlsec1-dev libxslt1-dev lynx nodejs npm ntp pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Dockerize will be useful to wait for mysql DB availability
ARG DOCKERIZE_VERSION=v0.6.1
RUN curl -L -o /tmp/dockerize.tar.gz https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz \
    && tar -C /usr/local/bin -xzvf /tmp/dockerize.tar.gz \
    && rm /tmp/dockerize.tar.gz

# Checkout edx-platform code: https://github.com/edx/edx-platform/commits/open-release/ironwood.master
# Because we are pulling from a branch, and not a tag, you should manually modify the
# date hash below to force clearing the docker cache.
ARG EDX_PLATFORM_REPOSITORY=https://github.com/morta10/edx-platform.git
ARG EDX_PLATFORM_VERSION=open-release/ironwood.master
ARG EDX_PLATFORM_VERSION_DATE=20200505
RUN mkdir -p /openedx/edx-platform && \
    echo "Pulling $EDX_PLATFORM_VERSION tag from $EDX_PLATFORM_REPOSITORY ($EDX_PLATFORM_VERSION_DATE)" && \
    git clone $EDX_PLATFORM_REPOSITORY --branch $EDX_PLATFORM_VERSION --depth 1 /openedx/edx-platform
WORKDIR /openedx/edx-platform

# Apply patches
# Allow SigV4 authentication for video uploads to S3 https://github.com/edx/edx-platform/pull/22080
RUN curl https://github.com/overhangio/edx-platform/commit/0d4f6cc3433013960b28e963c4094ef2a2a92f04.patch | git apply -
# Resolve missing tasks in CMS 
# https://github.com/edx/edx-platform/pull/21297/
# https://github.com/edx/edx-platform/pull/21305/
RUN curl https://github.com/edx/edx-platform/commit/adb2c672e4d17cc1c42bdc206a0051e0fa16b5be.patch | git apply -
RUN curl https://github.com/edx/edx-platform/commit/b7ecd80a2bef0d845c3bce97818e70fb3ed9e36d.patch | git apply -

# Download extra locales to /openedx/locale/contrib/locale
RUN cd /tmp \
    && curl -L -o openedx-i18n.tar.gz https://github.com/openedx/openedx-i18n/archive/ironwood.tar.gz \
    && tar xzf /tmp/openedx-i18n.tar.gz \
    && mkdir -p /openedx/locale/contrib \
    && mv openedx-i18n-ironwood/edx-platform/locale /openedx/locale/contrib \
    && rm -rf openedx-i18n*

# Install python requirements in a virtualenv
RUN virtualenv /openedx/venv
ENV PATH /openedx/venv/bin:${PATH}
ENV VIRTUAL_ENV /openedx/venv/
RUN pip install setuptools==39.0.1 pip==9.0.3
RUN pip install -r requirements/edx/base.txt

# Install patched version of ora2
RUN pip uninstall -y ora2 && \
    pip install git+https://github.com/overhangio/edx-ora2.git@2.2.0-patched#egg=ora2==2.2.0

# Install patched version of edx-oauth2-provider
RUN pip install git+https://github.com/overhangio/edx-oauth2-provider.git@1.2.3#egg=edx-oauth2-provider==1.2.3

# Install ironwood-compatible scorm xblock
RUN pip install "openedx-scorm-xblock<10.0.0,>=9.1.1"

# Install a recent version of nodejs
RUN nodeenv /openedx/nodeenv --node=8.9.3 --prebuilt
ENV PATH /openedx/nodeenv/bin:${PATH}

# Install nodejs requirements
ARG NPM_REGISTRY=https://registry.npmjs.org/
RUN npm set progress=false \
    && npm install --verbose --registry=$NPM_REGISTRY
ENV PATH ./node_modules/.bin:${PATH}

# Install private requirements: this is useful for installing custom xblocks.
COPY ./requirements/ /openedx/requirements
RUN cd /openedx/requirements/ \
  && touch ./private.txt \
  && pip install -r ./private.txt

# Create folder that will store *.env.json and *.auth.json files, as well as
# the tutor-specific settings files.
RUN mkdir -p /openedx/config ./lms/envs/tutor ./cms/envs/tutor
ENV CONFIG_ROOT /openedx/config
COPY settings/lms/*.py ./lms/envs/tutor/
COPY settings/cms/*.py ./cms/envs/tutor/

# Copy user-specific locales to /openedx/locale/user/locale and compile them
RUN mkdir -p /openedx/locale/user
COPY ./locale/ /openedx/locale/user/locale/
RUN cd /openedx/locale/user && \
    django-admin.py compilemessages -v1
# Compile i18n strings: in Ironwood, js locales are not properly compiled out of the box
# and we need to do a pass ourselves. Also, we need to compile the djangojs.js files for
# the downloaded locales.
RUN ./manage.py lms --settings=tutor.i18n compilejsi18n
RUN ./manage.py cms --settings=tutor.i18n compilejsi18n

# Copy scripts
COPY ./bin /openedx/bin
RUN chmod a+x /openedx/bin/*
ENV PATH /openedx/bin:${PATH}



# Collect production assets. By default, only assets from the default theme
# will be processed. This makes the docker image lighter and faster to build.
# Only the custom themes added to /openedx/themes will be compiled.
# Here, we don't run "paver update_assets" which is slow, compiles all themes
# and requires a complex settings file. Instead, we decompose the commands
# and run each one individually to collect the production static assets to
# /openedx/staticfiles.
ENV NO_PYTHON_UNINSTALL 1
RUN openedx-assets xmodule \
    && openedx-assets npm \
    && openedx-assets webpack --env=prod \
    && openedx-assets common
COPY ./themes/ /openedx/themes/
RUN openedx-assets themes \
    && openedx-assets collect --settings=tutor.assets

# Create a data directory, which might be used (or not)
RUN mkdir /openedx/data

# service variant is "lms" or "cms"
ENV SERVICE_VARIANT lms
ENV SETTINGS tutor.production

{{ patch("openedx-dockerfile") }}

# Entrypoint will set right environment variables
ENTRYPOINT ["docker-entrypoint.sh"]

# Run server
COPY gunicorn_conf.py /openedx/gunicorn_conf.py
EXPOSE 8000
CMD gunicorn -c /openedx/gunicorn_conf.py --name ${SERVICE_VARIANT} --bind=0.0.0.0:8000 --max-requests=1000 --access-logfile - ${SERVICE_VARIANT}.wsgi:application
