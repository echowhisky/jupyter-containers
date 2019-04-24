########################################
# Stage 1
# Operating System layer taking an Official base and adding the minimal pieces to set us up for additional Jupyter and nbGallery goodness.
#

# This is the Ubuntu version. It is loosly based on the Jupyter base-notebook dockerfile
ARG BASE_CONTAINER=ubuntu:bionic-20180526@sha256:c8c275751219dadad8fa56b3ac41ca6cb22219ff117ca98fe82b42f24e1ba64e
FROM $BASE_CONTAINER AS base

# The key things we'll do in this stage are:
# * add some desireable env variables
# * install some key applications we'll be needing
# * fixing the locale to en_US.UTF-8
# * creating a user account
# We're saving the miniconda installation and the conda package installs themselves for subsequent stages

# Args we'll use later for the user account setup
ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"

# Environment Variables 
ENV DEBIAN_FRONTEND=noninteractive \
    BASE_OS=ubuntu \
    SHELL=/bin/bash \
    NB_USER=$NB_USER \
    NB_UID=$NB_UID \
    NB_GID=$NB_GID \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8 \
    HOME=/home/$NB_USER \
    GRANT_SUDO=yes \
    CHOWN_HOME=yes \
    CHOWN_HOME_OPTS='-R' 

USER root

# Install OS dependencies
RUN apt-get update \
 && apt-get -yq dist-upgrade \
 && apt-get install -yq --no-install-recommends \
        curl \
        bzip2 \
        ca-certificates \
        sudo \
        locales \
        fonts-liberation \
        nginx \
# cleanup
 && rm -rf /var/lib/apt/lists/* \
# fix locale
 && echo "en_US.UTF-8 UTF-8" > /etc/locale.gen \
 && locale-gen \
# tweak the skeleton .bashrc
 && sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc \
# create user account
 && echo "### Creation of jovyan user account" \
 && echo "auth requisite pam_deny.so" >> /etc/pam.d/su \
 && sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers \
 && sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers \
 && useradd -m -s /bin/bash -N -u $NB_UID $NB_USER \
 && chmod g+w /etc/passwd 

########################################
# Stage 2
# This layer takes an OS layer and builds in the miniconda package and Jupyter libraries
#

FROM base as miniconda 

ENV CONDA_DIR=/opt/conda 
ENV PATH=$CONDA_DIR/bin:$PATH \
# !! key version numbers and other changeable information to watch out for
    MINICONDA_VERSION=4.5.12 \
    CONDA_VERSION=4.6.7 \
    MINICONDA_HASH=866ae9dff53ad0874e1d1a60b1ad1ef8 \
    TINI_VERSION=0.18.0 \
    NOTEBOOK_VERSION=5.7.8 \
    JUPYTERHUB_VERSION=0.9.6 \
    JUPYTERLAB_VERSION=0.35.4 \
    JUPYTERLAB_MANAGER_VERSION=0.38 \
    HUB_EXTENSION_VERSION=^0.12.0 

USER $NB_UID
COPY --chown=1000:100 util/* $CONDA_DIR/bin/

# Setup work directory for backward-compatibility
RUN mkdir /home/$NB_USER/work \
# Install conda as jovyan and check the md5 sum provided on the download site
 && echo "### Downloading miniconda" \
 && curl -sSL https://repo.continuum.io/miniconda/Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh -o /tmp/miniconda.sh \
 && echo `md5sum /tmp/miniconda.sh` \
 && echo "${MINICONDA_HASH} /tmp/miniconda.sh" | md5sum -c - \
 && echo "### Installing miniconda" \
 && bash /tmp/miniconda.sh -bfp $CONDA_DIR \
 && rm -rf /tmp/miniconda.sh \
 && echo "### Setting up and updating miniconda" \
 && $CONDA_DIR/bin/conda config --system --prepend channels conda-forge \
 && $CONDA_DIR/bin/conda config --system --set auto_update_conda false \
 && $CONDA_DIR/bin/conda config --system --set show_channel_urls true \
 && $CONDA_DIR/bin/conda install --quiet --yes conda="${CONDA_VERSION%.*}.*" \
 && $CONDA_DIR/bin/conda update --all --quiet --yes \
 && conda clean -tipsy \
 && rm -rf /home/$NB_USER/.cache/yarn \
 && fix-permissions $CONDA_DIR \
 && fix-permissions /home/$NB_USER

# Install all the conda things needed
RUN conda install --quiet --yes \
        "tini=$TINI_VERSION" \
        "notebook=$NOTEBOOK_VERSION" \
        "jupyterhub=$JUPYTERHUB_VERSION" \
        "jupyterlab=$JUPYTERLAB_VERSION" \
        "make" \
        "jupyter_nbextensions_configurator" \
        "jupyter_dashboards" \
        "ipywidgets" \
        "ipyleaflet" \
        "qgrid" \
 && conda clean -tipsy \
 && conda clean --all --yes \
 && conda list tini | grep tini | tr -s ' ' | cut -d ' ' -f 1,2 >> $CONDA_DIR/conda-meta/pinned \
 && jupyter labextension install \
        @jupyterlab/hub-extension@$HUB_EXTENSION_VERSION \
        @jupyter-widgets/jupyterlab-manager@$JUPYTERLAB_MANAGER_VERSION \
        qgrid \
        jupyter-leaflet \
 && npm cache clean --force \
 && jupyter notebook --generate-config \
 && rm -rf $CONDA_DIR/share/jupyter/lab/staging \
 && rm -rf /home/$NB_USER/.cache/yarn \
 && fix-permissions $CONDA_DIR \
 && fix-permissions /home/$NB_USER

########################################
# Stage 3
# 
# Any final pieces are added. Preference files, etc.

FROM base  

ENV CONDA_DIR=/opt/conda 
ENV PATH=$CONDA_DIR/bin:$PATH \
    MINICONDA_VERSION=$MINICONDA_VERSION \
    CONDA_VERSION=$CONDA_VERSION \
    JUPYTER=$CONDA_DIR/bin/jupyter

USER $NB_UID

COPY --chown=1000:100 --from=miniconda $CONDA_DIR $CONDA_DIR
COPY --chown=1000:100 --from=miniconda $HOME $HOME

RUN conda install --quiet --yes \
        'pandas' \
 && conda clean -tipsy \
 && fix-permissions $CONDA_DIR 

# set startpoints
EXPOSE 8888
WORKDIR $HOME

# Configure container startup
ENTRYPOINT ["tini", "-g", "--"]
CMD ["start-notebook.sh"]

USER $NB_UID
########################################
