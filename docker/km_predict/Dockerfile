# ==== Build ====
FROM ubuntu:22.04 AS builder

ENV LANG=C.UTF-8 LC_ALL=C.UTF-8

WORKDIR /home

# Install pre-dependencies
RUN apt-get update --fix-missing && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
	software-properties-common git
# Install build tools and Mapserver dependencies.
RUN apt-get update --fix-missing && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
    cmake gcc g++ python3-pip libgdal-dev gdal-bin libexpat1-dev libgraphicsmagick++1-dev libcppunit-dev

# Install cget
RUN pip3 install cget

# Get cm-vsm
RUN git clone --recurse-submodules https://github.com/kappazeta/cm-vsm.git && \
    cd cm-vsm && git checkout v0.3.5
# Install dependencies via cget
RUN cd /home/cm-vsm/vsm/build && \
    cget install -DJSON_BuildTests=Off
# Prepare for build
RUN cd /home/cm-vsm/vsm/build && \
    cmake ..
# Build
RUN cd /home/cm-vsm/vsm/build && \
    make && \
	make install

# ==== Runtime ====
FROM ubuntu:22.04 as runtime

ENV LANG=C.UTF-8 LC_ALL=C.UTF-8
ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib/:/usr/local/lib/x86_64-linux-gnu/

WORKDIR /home

# Install system dependencies.
RUN apt-get update --fix-missing && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
    lsb-release wget unzip vim-tiny libgdal30

# Clean up after ourselves.
RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Copy binaries.
COPY --from=builder /usr/bin/ /usr/bin/
COPY --from=builder /usr/lib/ /usr/lib/
RUN ldconfig

# Get cm-predict
RUN mkdir -p /home/km_predict_weights && \
    wget -q http://kappamask.s3-website.eu-central-1.amazonaws.com/model_weights/2022-09-13/l1c_deeplabv3plus.hdf5 -O /home/km_predict_weights/l1c_deeplabv3plus.hdf5
RUN wget -q http://kappamask.s3-website.eu-central-1.amazonaws.com/model_weights/2022-09-13/l2a_deeplabv3plus.hdf5 -O /home/km_predict_weights/l2a_deeplabv3plus.hdf5
RUN git clone --recurse-submodules https://github.com/kappazeta/km_predict.git && cd km_predict && git checkout v2.0
RUN mv /home/km_predict_weights /home/km_predict/weights

# Install MiniConda.
RUN wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O /tmp/Miniconda3-latest-Linux-x86_64.sh
RUN bash /tmp/Miniconda3-latest-Linux-x86_64.sh -b -p /opt/conda
RUN rm /tmp/Miniconda3-latest-Linux-x86_64.sh
RUN chmod a+rx /opt/conda/etc/profile.d/conda.sh
RUN ln -s /opt/conda/etc/profile.d/conda.sh /etc/profile.d/conda.sh
RUN /opt/conda/bin/conda install -c conda-forge mamba
ENV PATH="${PATH}:/opt/conda/bin"

# Install dependencies
RUN cd /home/km_predict && \
    mamba env create -p /opt/conda/envs/km_predict -f environment.yml && \
    mamba clean --all
RUN bash -c 'source /etc/profile.d/conda.sh && conda activate km_predict && pip3 install boto3 sentinelhub[AWS]'

# Download AWS CLI.
RUN wget "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -O /home/awscliv2.zip
# Install AWS CLI.
RUN unzip /home/awscliv2.zip && \
    rm /home/awscliv2.zip && \
    ./aws/install

COPY get_s3.py /home/
COPY km_s3.sh /home/
RUN chmod +x /home/km_s3.sh && \
    ln -s /home/km_s3.sh /usr/local/bin/km_s3

ENTRYPOINT ["km_s3"]
#ENTRYPOINT ["sleep", "infinity"]
