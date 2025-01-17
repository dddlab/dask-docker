ARG base

FROM $base/base-notebook:lab

ARG release
ARG python

USER root

ENV PYTHON_VERSION=${python}
ENV DASK_VERSION=${release}

RUN apt-get update \
    && apt-get install -yq --no-install-recommends curl graphviz git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# We don't use TARGETARCH so as to support non-buildkit builds
RUN MAGICARCH=$(dpkg --print-architecture) && \
    curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/${MAGICARCH}/kubectl" && \
    mkdir -p /usr/local/bin && \
    mv ./kubectl /usr/local/bin/kubectl && \
    chmod +x /usr/local/bin/kubectl && \
    kubectl version --client

USER $NB_USER

RUN conda install mamba -n base -c conda-forge --yes
RUN mamba install --yes python=${PYTHON_VERSION} nomkl \
    && mamba install --yes \
    python-blosc \
    cytoolz \
    dask==${DASK_VERSION} \
    lz4 \
    numpy==1.21.1 \
    pandas==1.3.0 \
    ipywidgets \
    cachey \
    streamz \
    dask-labextension>=5 \
    && mamba clean -tipsy \
    && jupyter lab clean \
    && jlpm cache clean \
    && npm cache clean --force \
    && find /opt/conda/ -type f,l -name '*.a' -delete \
    && find /opt/conda/ -type f,l -name '*.pyc' -delete \
    && find /opt/conda/ -type f,l -name '*.js.map' -delete \
    && (find /opt/conda/lib/python*/site-packages/bokeh/server/static -type f,l -name '*.js' -not -name '*.min.js' -delete || echo "no bokeh static files to cleanup") \
    && rm -rf /opt/conda/pkgs

USER root

# Create the /opt/app directory, and assert that Jupyter's NB_UID/NB_GID values
# haven't changed.
RUN mkdir /opt/app \
    && if [ "$NB_UID" != "1000" ] || [ "$NB_GID" != "100" ]; then \
    echo "Jupyter's NB_UID/NB_GID changed, need to update the Dockerfile"; \
    exit 1; \
    fi

RUN \
    # download R studio
    curl --silent -L --fail \
        https://s3.amazonaws.com/rstudio-ide-build/server/bionic/amd64/rstudio-server-1.2.1578-amd64.deb > /tmp/rstudio.deb && \
    \
    # install R studio
    apt-get update && \
    apt-get install -y --no-install-recommends /tmp/rstudio.deb && \
    rm /tmp/rstudio.deb && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy over the example as NB_USER. Unfortuantely we can't use $NB_UID/$NB_GID
# in the `--chown` statement, so we need to hardcode these values.
COPY --chown=1000:100 examples/ /home/$NB_USER/examples
COPY prepare.sh /usr/bin/prepare.sh

USER ${NB_USER}

RUN pip install \ 
        jupyter-server-proxy==1.6.0 \
        jupyter-rsession-proxy==1.2.0 && \
    jupyter labextension install @jupyterlab/server-proxy && \
    \
    pip install \
        jupyterlab-git==0.32.2 \
        nbgitpuller==0.10.1 && \
    jupyter serverextension enable --sys-prefix nbgitpuller && \
    jupyter lab build

USER root

ENTRYPOINT ["tini", "--", "/usr/bin/prepare.sh"]
