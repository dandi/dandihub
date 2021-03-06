FROM golang:1.15.2 as builder

ARG SINGULARITY_COMMITISH="v3.7.4"

WORKDIR $GOPATH/src/github.com/sylabs
RUN apt-get update && apt-get install -y \
    build-essential \
    uuid-dev \
    libgpgme-dev \
    squashfs-tools \
    libseccomp-dev \
    wget \
    pkg-config \
    git \
    cryptsetup-bin
RUN git clone https://github.com/sylabs/singularity.git \
    && cd singularity \
    && git checkout "$SINGULARITY_COMMITISH" \
    && ./mconfig -p /usr/local/singularity \
    && cd builddir \
    && make \
    && make install

FROM jupyter/datascience-notebook:2492e8be8a11

USER root
COPY --from=builder /usr/local/singularity /usr/local/singularity
ENV PATH="/usr/local/singularity/bin:$PATH"
RUN apt-get update && apt-get install -y ca-certificates libseccomp2 \
   squashfs-tools fuse s3fs netbase less parallel tmux vim \
   && rm -rf /tmp/*

RUN curl --silent --show-error "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
  -o "awscliv2.zip" && unzip awscliv2.zip \
  && ./aws/install && rm -rf ./aws awscliv2.zip

USER $NB_USER

# Install Allen SDK
RUN conda create -n allen -c conda-forge python=3.8 pip ipykernel \
  && /opt/conda/envs/allen/bin/pip install --no-cache-dir ipfx \
  && conda clean --all -f -y && rm -rf /tmp/*

# Install DANDI env
RUN conda create -n dandi -c conda-forge python=3.8 pip ipykernel \
  && /opt/conda/envs/dandi/bin/pip install --no-cache-dir dandi \
  nibabel nilearn pybids zarr spikeinterface neo pydra nwbwidgets jupytext \
  plotly itkwidgets==0.32.0 \
  && conda clean --all -f -y && rm -rf /tmp/*

RUN conda install --yes datalad rclone \
  && wget --quiet https://raw.githubusercontent.com/DanielDent/git-annex-remote-rclone/v0.6/git-annex-remote-rclone \
  && chmod +x git-annex-remote-rclone && mv git-annex-remote-rclone /opt/conda/bin \
  && pip install --no-cache-dir jupytext nbgitpuller datalad-container \
     datalad-osf \
  && conda clean --all -f -y && rm -rf /tmp/*

RUN jupyter labextension install @jupyter-widgets/jupyterlab-manager \
  jupyter-matplotlib jupyterlab-datawidgets itkwidgets@0.32.0 jupyterlab-plotly \
  plotlywidget jupyter-threejs --no-build \
  && export NODE_OPTIONS=--max-old-space-size=4096 \
  && jupyter lab build && \
     jupyter lab clean && \
     jlpm cache clean && \
     npm cache clean --force && \
     rm -rf $HOME/.node-gyp && \
     rm -rf $HOME/.local && rm -rf /tmp/*

# Install NWBstream env
RUN conda create -n ros3 -c conda-forge python=3.8 hdf5 h5py=3.2 ipykernel pip \
  numpy matplotlib \
  && /opt/conda/envs/ros3/bin/pip install --no-cache-dir \
  git+https://github.com/hdmf-dev/hdmf.git@enh/h5py3 \
  && git clone --recurse-submodules --depth 1 https://github.com/NeurodataWithoutBorders/pynwb.git \
  && sed -i s/'h5py>=2.9,<3'/'h5py>3'/g pynwb/setup.py \
  && /opt/conda/envs/ros3/bin/pip install --no-cache-dir pynwb/. \
  && rm -rf pynwb \
  && /opt/conda/envs/ros3/bin/pip install --no-cache-dir \
  nwbwidgets jupytext plotly itkwidgets==0.32.0 \
  && conda clean --all -f -y && rm -rf /tmp/*

# Install jupyter server proxy and desktop
USER root

RUN apt-get -y update \
   && apt-get install -y dbus-x11 \
       firefox \
       xfce4 \
       xfce4-panel \
       xfce4-session \
       xfce4-settings \
       xorg \
       xubuntu-icon-theme \
    && rm -rf /tmp/*

# Remove light-locker to prevent screen lock
RUN wget --quiet 'https://sourceforge.net/projects/turbovnc/files/2.2.5/turbovnc_2.2.5_amd64.deb/download' -O turbovnc_2.2.5_amd64.deb && \
       apt-get install -y -q ./turbovnc_2.2.5_amd64.deb && \
       apt-get remove -y -q light-locker && \
       rm ./turbovnc_2.2.5_amd64.deb && \
       ln -s /opt/TurboVNC/bin/* /usr/local/bin/ \
    && rm -rf /tmp/*

# apt-get may result in root-owned directories/files under $HOME
RUN mkdir /opt/extras && chown -R $NB_UID:$NB_GID $HOME /opt/extras

USER $NB_USER

RUN cd /opt/extras && \
    git clone --depth 1 https://github.com/jupyterhub/jupyter-remote-desktop-proxy.git && \
    cd jupyter-remote-desktop-proxy && \
    conda env update -n base --file environment.yml && \
    cd /opt && rm -rf /opt/extras/* && \
    conda clean --all -f -y && rm -rf /tmp/*

RUN /opt/conda/envs/allen/bin/python -m ipykernel install --user --name allen \
    --display-name="Allen SDK" \
    && /opt/conda/envs/dandi/bin/python -m ipykernel install --user --name dandi \
    --display-name="DANDI" \
    && /opt/conda/envs/ros3/bin/python -m ipykernel install --user --name ros3 \
    --display-name="NWBstream"

