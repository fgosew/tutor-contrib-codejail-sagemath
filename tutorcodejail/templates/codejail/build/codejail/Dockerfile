FROM docker.io/ubuntu:22.04 as minimal

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update -y && \
    apt-get install --no-install-recommends -y \
        make \
        build-essential \
        wget \
        curl \
        llvm \
        git \
        apt-transport-https \
        ca-certificates \
        language-pack-en \
        update-ca-certificates && \
    apt-get -y clean && \
    apt-get -y autoremove && \
    rm -rf /var/lib/apt/lists/*
ENV LC_ALL en_US.UTF-8

FROM minimal as python

SHELL ["/bin/bash", "-c"]
USER root

RUN apt-get update -y && \
    apt-get install --no-install-recommends -y \
        libssl-dev \
        zlib1g-dev \
        libbz2-dev \
        libreadline-dev \
        libsqlite3-dev \
        wget \
        curl \
        llvm \
        libncurses5-dev \
        xz-utils \
        tk-dev \
        libxml2-dev \
        libxmlsec1-dev \
        libffi-dev \
        liblzma-dev \
        python3-openssl \
        git && \
    apt-get -y clean && \
    apt-get -y autoremove && \
    rm -rf /var/lib/apt/lists/*

ENV HOME /root
WORKDIR /root
ENV PYENV_ROOT /opt/pyenv

#needed or the https request / tls in general will fail
RUN update-ca-certificates

# Install pyenv and pyenv-virtualenv
ENV PYENV_RELEASE_VERSION=2.3.12
RUN git clone --depth 1 https://github.com/pyenv/pyenv.git \
        --branch "v${PYENV_RELEASE_VERSION}" \
        --single-branch \
        ${PYENV_ROOT} && \
    pushd ${PYENV_ROOT} && \
    src/configure && \
    make -C src && \
    popd && \
    sed -i '/^# ~.*/a export PYENV_ROOT="${PYENV_ROOT}"' ~/.profile && \
    sed -i '/^export.*/a export PATH="${PYENV_ROOT}/bin:${PATH}"' ~/.profile && \
    sed -i '/^export PATH.*/a \\n# Place pyenv shims on path\nif [[ ":${PATH}:" != *":$(pyenv root)/shims:"* ]]; then\n  eval "$(pyenv init --path)"\nfi' ~/.profile && \
    printf '\neval "$(pyenv init -)"\n' >> ~/.bashrc && \
    . ~/.profile && \
    git clone --depth 1 https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv && \
    git clone https://github.com/s1341/pyenv-alias.git $(pyenv root)/plugins/pyenv-alias && \
    printf '\n# Place pyenv-virtualenv shims on path\nif [[ ":${PATH}:" != *":$(pyenv root)/plugins/pyenv-virtualenv/shims:"* ]]; then\n  eval "$(pyenv init -)"\nfi\n' >> ~/.profile && \
    printf '\n# Place pyenv shims on path\nif [[ ":${PATH}:" != *":$(pyenv root)/shims:"* ]]; then\n  eval "$(pyenv init --path)"\nfi\n' >> ~/.bashrc && \
    printf '# Place pyenv-virtualenv shims on path\nif [[ ":${PATH}:" != *":$(pyenv root)/plugins/pyenv-virtualenv/shims:"* ]]; then\n  eval "$(pyenv virtualenv-init -)"\nfi\n' >> ~/.bashrc && \
    cp ~/.profile ~/.bash_profile && \
    sed -i 's/.profile/.bash_profile/' ~/.bash_profile

ARG CODEJAILSERVICE_PYTHON_VERSION=3.8.6

RUN . "${HOME}/.bash_profile" && \
    pyenv install "${CODEJAILSERVICE_PYTHON_VERSION}" 

RUN $PYENV_ROOT/versions/$CODEJAILSERVICE_PYTHON_VERSION/bin/python -m venv /openedx/venv
#we don't install conda here yet as this would waste space due to the stages being reused later on.
#installing conda and sage will happen in the production stage to keep the needed space to a minimum during
#setup

###### Codejail service code

FROM minimal as code

RUN git clone https://github.com/eduNEXT/codejailservice.git --branch {{ CODEJAIL_VERSION }} --depth 1 /openedx/codejailservice
WORKDIR /openedx/codejailservice
#RUN git checkout ${CODEJAIL_VERSION}

###### Install python requirements in virtualenv
FROM python as codejailservice-python-requirements

ENV PATH /openedx/venv/bin:${PATH}
ENV VIRTUAL_ENV /openedx/venv/

COPY --from=code /openedx/codejailservice /openedx/codejailservice
WORKDIR /openedx/codejailservice
RUN pip3 install -r requirements/base.txt
RUN pip3 install uwsgi==2.0.21

###### Install python requirements in virtualenv
FROM python as sandbox-python-requirements

ENV PATH /sandbox/venv/bin:${PATH}
ENV VIRTUAL_ENV /sandbox/venv/

WORKDIR /var/tmp
RUN mkdir -p common/lib/

COPY --from={{ DOCKER_IMAGE_OPENEDX }} /openedx/edx-platform/requirements/edx-sandbox/py38.txt py38.txt
RUN pip3 install -r py38.txt

##### Prod image
FROM minimal as production

SHELL ["/bin/bash", "-c"]
USER root

ENV PYENV_ROOT /opt/pyenv
ARG SANDBOX_PYTHON_VERSION={{ CODEJAIL_SANDBOX_PYTHON_VERSION }}

{% if CONDA_VERSION is defined %}
ENV CONDA_VERSION={{ CONDA_VERSION }} 
{% else %}
ENV CONDA_VERSION=miniconda3-latest
{% endif %}

# Setup sandbox
ENV SANDBOX_ENV=/sandbox/venv
RUN mkdir /sandbox
RUN groupadd -r sandbox && useradd -m -r -g sandbox sandbox && chown -R sandbox:sandbox /sandbox

# Install system requirements
RUN apt update && \
    apt install -y sudo

COPY --chown=sandbox:sandbox --from=python $PYENV_ROOT $PYENV_ROOT
COPY --from=python /root/.bash_profile /root/.bash_profile
COPY --from=python /root/.bashrc /root/.bashrc
COPY --from=python /root/.bash_profile /home/sandbox/.bash_profile
COPY --from=python /root/.bashrc /home/sandbox/.bashrc

RUN chown sandbox:sandbox /home/sandbox/.bash*

#we do this complicated move due to the performance of chown. Instead of chowning later
#roughly 5 GB of data we install directly as user 'sandbox'. 
USER sandbox

RUN . "/home/sandbox/.bash_profile" && \
    pyenv init - && \
    pyenv init --path && \
    pyenv virtualenv-init - && \
    pyenv install "${CONDA_VERSION}" && \
    pyenv shell "${CONDA_VERSION}" && \  
    conda init && \
    conda config --set auto_activate_base false && \
    conda config --env --add channels conda-forge && \
    conda config --env --set channel_priority strict && \
    conda install -y mamba && \
    #directly install conda's sage installation matching the python version of the given sandbox python version
    #to the sandbox folder. 
    mamba create -y --prefix=/sandbox/venv/ sage python=${SANDBOX_PYTHON_VERSION}

USER root

# Allows you to add extra pip requirements to your codejail sandbox.
{% if CODEJAIL_EXTRA_PIP_REQUIREMENTS is defined %}
{% for extra_requirements in CODEJAIL_EXTRA_PIP_REQUIREMENTS %}RUN --mount=type=cache,target=/openedx/.cache/pip,sharing=shared /sandbox/venv/bin/pip install '{{ extra_requirements }}'
{% endfor %}
{% endif %}

#requirement which was missing from the codejail which resulted in an unknown module error
RUN --mount=type=cache,target=/openedx/.cache/pip,sharing=shared /sandbox/venv/bin/pip install random2

COPY --from=code /openedx/codejailservice /openedx/codejailservice
COPY --from=codejailservice-python-requirements /openedx/venv /openedx/venv

#clean up the sandbox pyenv and copy the previously generated one from the python stage
#this also removes the conda installation, which is not needed anymore and would waste 
#roughly 8 GB of storage if not deleted
RUN rm -rf ${PYENV_ROOT}/
COPY --from=python $PYENV_ROOT $PYENV_ROOT

#delete files that might fail due to path / values not being available
RUN rm /home/sandbox/.bashrc
RUN rm /home/sandbox/.bash_profile

ENV PATH /openedx/venv/bin:${PATH}
ENV VIRTUAL_ENV /openedx/venv/
WORKDIR /openedx/codejailservice

EXPOSE 8550
CMD uwsgi \
    --http 0.0.0.0:8550 \
    --thunder-lock \
    --single-interpreter \
    --enable-threads \
    --processes=${UWSGI_WORKERS:-2} \
    --buffer-size=8192 \
    --max-requests=1000 \
    --wsgi-file /openedx/codejailservice/wsgi.py
