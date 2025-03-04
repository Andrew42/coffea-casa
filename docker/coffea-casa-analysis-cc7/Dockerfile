FROM coffeateam/coffea-dask-cc7:0.7.12-fastjet-3.3.4.0rc9-g0aa0af3

USER root
LABEL maintainer="Oksana Shadura <ksu.shadura@gmail.com>"
# Jupyterhub
ARG NB_USER="cms-jovyan"
ARG NB_UID="6440"
ARG NB_GID="11265"
# Tag
ARG TAG="2021.12.06dev0"
# XCACHE
ARG XCACHE_HOST="red-xcache1.unl.edu"
ARG CERT_DIR="/etc/cmsaf-secrets"
# XCACHE settings for Servicex
ARG CACHE_PREFIX="red-xcache1.unl.edu"
ARG BEARER_TOKEN_FILE=$CERT_DIR"/xcache_token"

# Hack for GH Actions
ARG GITHUB_ACTIONS="false"

# Configure environment
ENV CONDA_DIR /usr/local
ENV SHELL /bin/bash
ENV NB_USER $NB_USER
ENV USER $NB_USER
ENV NB_UID $NB_UID
ENV NB_GID $NB_GID
ENV HOME /home/$NB_USER
ENV PATH "${CONDA_DIR}/bin/:$PATH"
ENV TAG $TAG
ENV XCACHE_HOST $XCACHE_HOST
ENV CACHE_PREFIX $CACHE_PREFIX
ENV CERT_DIR $CERT_DIR
ENV BEARER_TOKEN_FILE $BEARER_TOKEN_FILE
ENV LC_ALL en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US.UTF-8

# Install all OS dependencies for notebook server that starts but lacks all
# features (e.g., download as all possible file formats)
RUN yum -y install http://opensource.wandisco.com/centos/7/git/x86_64/wandisco-git-release-7-2.noarch.rpm   \
 && yum -y update \
 && yum -y group install "Development Tools" \
 && yum -y install \
    wget \
    epel-release \
    gettext \
    bzip2 \
    ca-certificates \
    sudo \
    langpacks-en \
    glibc-all-langpacks \
    liberation-fonts \
    wget \
    nss_wrapper \
    openssl \
    emacs \
    git \
    unzip \
    nano \
    vim  \
    curl \
    net-tools \
    libXext \
    libSM \
    libXrender \
    nc \
    openssh-clients \
    tzdata \
    unzip \
    gdb \
    && yum clean all && rm -rf /var/cache/yum

# Copy a script that we will use to correct permissions after running certain commands
COPY fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

# Enable prompt color in the skeleton .bashrc before creating the default NB_USER
# hadolint ignore=SC2016
RUN sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc && \
   # Add call to conda init script see https://stackoverflow.com/a/58081608/4413446
   echo 'eval "$(command conda shell.bash hook 2> /dev/null)"' >> /etc/skel/.bashrc

# Create NB_USER with name jovyan user with given UID and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    groupadd -r "${NB_USER}" --gid "$NB_GID" && \
    useradd -l -m -s /bin/bash -N -u "${NB_UID}" "${NB_USER}" && \
    mkdir -p "${CONDA_DIR}" && \
    mkdir -p "${CERT_DIR}" && \
    mkdir -p /var/lib/condor && \
    chown -R "${NB_USER}:${NB_GID}" "${CONDA_DIR}" && \
    chown -R "${NB_USER}:${NB_GID}" "${HOME}" && \
    chmod g+w /etc/passwd && \
    fix-permissions "${HOME}" && \
    fix-permissions "${CONDA_DIR}"

USER $NB_USER
WORKDIR $HOME
# Setup work directory for backward-compatibility
RUN mkdir $HOME/work && \
    mkdir -p $HOME/.local && \
    mkdir -p $HOME/.condor/tokens.d && \
    fix-permissions $HOME

# Dask dependencies and HTCondor
# https://anaconda.org/conda-forge/htcondor
RUN mamba install --yes \
    -c conda-forge \
    htcondor \
    oidc-agent \
    xgboost \
    pyhf \
    vector \
    hist \
    mplhep \
    iminuit \
    cmake

#
RUN pip install --no-cache-dir coffea[servicex] \
    # for running/restarting Dask worker in sidecar container
    supervisor \
    correctionlib \
    func_adl_uproot \
    cabinetry

# ------- xrootd-authz-plugin -------------------------------
#RUN cd /tmp && \
#    git clone https://github.com/bbockelm/xrdcl-authz-plugin.git && \
#    cd xrdcl-authz-plugin && \
#    mkdir build && \
#    cd  build && \
#    cmake /tmp/xrdcl-authz-plugin -DCMAKE_INSTALL_PREFIX=${CONDA_DIR} && \
#    make && \
#    make install

USER root
# Setup supervisord files
COPY supervisord.conf  /etc/supervisor/

# Add HTCondor configuration files
COPY condor_config /etc/condor/
# Copy configuration files (currently all, to be optimised)
COPY config.d  /etc/condor/config.d/

# Setup HTCondor user/group and change group for user $NB_USER
# Fix error (submitting jobs as user/group 0 (root) is not allowed for security reasons) and
# it configured from kubernetes side and updated in docker container to match it
RUN groupadd -r condor && \
    useradd -r -g condor -d /var/lib/condor -s /sbin/nologin condor

# Patching uproot (broken xrootd-authz-plugin with xrootd > 5.2.0 and uproot.MultithreadedXRootDSource)
COPY uproot_xrd_source.patch ${CONDA_DIR}/lib/python3.8/site-packages/uproot
RUN cd ${CONDA_DIR}/lib/python3.8/site-packages/uproot && patch < uproot_xrd_source.patch

# FIXME: merge PRs open in distributed.git (oshadura)
# Distributed: we need to install patched version of distributed version
COPY distributed ${CONDA_DIR}/lib/python3.8/site-packages/distributed
RUN cd ${CONDA_DIR}/lib/python3.8/site-packages/distributed && \
    patch -p2 < 0001-Patch-from-bbockelman-adaptive-scaling.patch && \
    patch -p2 < 0002-Allow-scheduler-to-preserve-worker-hostnames.patch && \
    patch -p2 < 0003-Activate-patch.patch && \
    patch -p2 < 0004-Add-possibility-to-setup-external_adress-for-schedul.patch
    # && patch -p2 < 0005-Add-patch-from-John-Thiltges.patch

# xcache setup
#ENV XRD_PLUGINCONFDIR="${CONDA_DIR}/etc/xrootd/client.plugins.d/"
ENV LD_LIBRARY_PATH="${CONDA_DIR}/lib/:$LD_LIBRARY_PATH"
#ENV XRD_PLUGIN="${CONDA_DIR}/lib/libXrdClAuthzPlugin.so"
ENV PATH="${CONDA_DIR}/bin/:$PATH"

# FIXME: we have a wrong path, let's make a link.
# cms-jovyan@jupyter-oksana-2eshadura-40cern-2ech:~$ echo $PATH
# /opt/conda/condabin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
RUN rm -rf ${CONDA_DIR}/condabin && ln -s ${CONDA_DIR}/bin ${CONDA_DIR}/condabin

# Cleanup
RUN rm -rf /tmp/* \
    && rm -rf $HOME/.cache/.pip/* \
    && mamba clean -tipsy \
    && find ${CONDA_DIR} -type f -name '*.a' -delete \
    && find ${CONDA_DIR} -type f -name '*.pyc' -delete \
    && find ${CONDA_DIR} -type f -name '*.js.map' -delete \
    && rm -rf ${CONDA_DIR}/pkgs

# Mkdir CVMFS mount directory.  Mount done upstream in configs. Harmless
# if not utilized, crucial if CVMFS is desired
RUN mkdir /cvmfs


# Prepare HTCondor spec. environment and execute dask-worker command
# If we using this container as a sidecar, we don't setup any HTCondor spec. environment
# nor and execute dask-worker command
ADD prepare-env.sh /usr/local/bin/
RUN chmod ugo+x /usr/local/bin/prepare-env.sh

USER $NB_USER
ENTRYPOINT ["tini", "-g", "--", "/usr/local/bin/prepare-env.sh"]
