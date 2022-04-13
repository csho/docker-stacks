# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.
ARG OWNER=jupyter
ARG BASE_CONTAINER=$OWNER/scipy-notebook
FROM $BASE_CONTAINER

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# Spark dependencies
# Default values can be overridden at build time
# (ARGS are in lower case to distinguish them from ENV)
ARG spark_version="3.2.1"
ARG hadoop_version="3.2.3"
ARG hive_version="3.1.3"
ARG spark_checksum="0923B887BFFE9CE984B41E730A0059D563D0EE429F4E8C74BE2DF98D0B441919EFF4CC3C43D79B131D3B914139DF4833AEE75280889643690E8C14A287552B40"
ARG hadoop_checksum="46bfbbec3a80dec4d4775e807a981260cac8719964d45663eb8215adcd716b90ef3225d4fe781bcce02ea15e5d2155733bfe0f41f985f2b15297f61abbeb2979"
ARG hive_checksum="0c9b6a6359a7341b6029cc9347435ee7b379f93846f779d710b13f795b54bb16"
ARG openjdk_version="11"

ENV APACHE_SPARK_VERSION="${spark_version}" \
    HADOOP_VERSION="${hadoop_version}" \
    HIVE_VERSION="${hive_version}"

RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    "openjdk-${openjdk_version}-jre-headless" \
    ca-certificates-java && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
# On amd64. change it on x86 or arm64.
# ENV JAVA_HOME="/usr/lib/jvm/java-11-openjdk-arm64"
ENV JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"

#download and install hadoop
WORKDIR /tmp
RUN wget -q "https://dlcdn.apache.org/hadoop/common/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}.tar.gz" && \
echo "${hadoop_checksum} *hadoop-${HADOOP_VERSION}.tar.gz" | sha512sum -c - && \
tar xzf "hadoop-${HADOOP_VERSION}.tar.gz" -C /usr/local --owner root --group root --no-same-owner && \
rm "hadoop-${HADOOP_VERSION}.tar.gz"

#Configure Hadoop
WORKDIR /usr/local
ENV HADOOP_HOME=/usr/local/hadoop-${HADOOP_VERSION}

# Spark installation
WORKDIR /tmp
RUN wget -q "https://archive.apache.org/dist/spark/spark-${APACHE_SPARK_VERSION}/spark-${APACHE_SPARK_VERSION}-bin-without-hadoop.tgz" && \
    echo "${spark_checksum} *spark-${APACHE_SPARK_VERSION}-bin-without-hadoop.tgz" | sha512sum -c - && \
    tar xzf "spark-${APACHE_SPARK_VERSION}-bin-without-hadoop.tgz" -C /usr/local --owner root --group root --no-same-owner && \
    rm "spark-${APACHE_SPARK_VERSION}-bin-without-hadoop.tgz"

WORKDIR /usr/local

# Configure Spark
ENV SPARK_HOME=/usr/local/spark-${APACHE_SPARK_VERSION}-bin-without-hadoop
ENV SPARK_OPTS="--driver-java-options=-Xms1024M --driver-java-options=-Xmx524288M --driver-java-options=-Dlog4j.logLevel=warning" 
ENV SPARK_DIST_CLASSPATH="${HADOOP_HOME}/etc/hadoop:${HADOOP_HOME}/share/hadoop/common/lib/*:${HADOOP_HOME}/share/hadoop/common/*:\
${HADOOP_HOME}/share/hadoop/hdfs:${HADOOP_HOME}/share/hadoop/hdfs/lib/*:${HADOOP_HOME}/share/hadoop/hdfs/*:\
${HADOOP_HOME}/share/hadoop/mapreduce/lib/*:${HADOOP_HOME}/share/hadoop/mapreduce/*:${HADOOP_HOME}/share/hadoop/yarn:\
${HADOOP_HOME}/share/hadoop/yarn/lib/*:${HADOOP_HOME}/share/hadoop/yarn/*"

RUN ln -s "spark-${APACHE_SPARK_VERSION}-bin-without-hadoop" spark && \
    # Add a link in the before_notebook hook in order to source automatically PYTHONPATH
    mkdir -p /usr/local/bin/before-notebook.d && \
    ln -s "${SPARK_HOME}/sbin/spark-config.sh" /usr/local/bin/before-notebook.d/spark-config.sh

# Fix Spark installation for Java 11 and Apache Arrow library
# see: https://github.com/apache/spark/pull/27356, https://spark.apache.org/docs/latest/#downloading
RUN cp -p "${SPARK_HOME}/conf/spark-defaults.conf.template" "${SPARK_HOME}/conf/spark-defaults.conf" && \
    echo 'spark.driver.extraJavaOptions -Dio.netty.tryReflectionSetAccessible=true' >> "${SPARK_HOME}/conf/spark-defaults.conf" && \
    echo 'spark.executor.extraJavaOptions -Dio.netty.tryReflectionSetAccessible=true' >> "${SPARK_HOME}/conf/spark-defaults.conf"

# Download and install Hive 
WORKDIR /tmp
RUN wget -q "https://downloads.apache.org/hive/hive-${HIVE_VERSION}/apache-hive-${HIVE_VERSION}-bin.tar.gz" && \
    echo "${hive_checksum} *apache-hive-${HIVE_VERSION}-bin.tar.gz" | sha256sum -c - && \
    tar xzf "apache-hive-${HIVE_VERSION}-bin.tar.gz" -C /usr/local --owner root --group root --no-same-owner && \
    rm "apache-hive-${HIVE_VERSION}-bin.tar.gz"

# Configure Hive
ENV HIVE_HOME=/usr/local/apache-hive-${HIVE_VERSION}-bin
ENV PATH="${PATH}:${SPARK_HOME}/bin:${HADOOP_HOME}/bin:${HIVE_HOME}/bin"
RUN rm $HIVE_HOME/lib/guava*.jar && \
    cp $HADOOP_HOME/share/hadoop/common/lib/guava*.jar $HIVE_HOME/lib/
# Configure IPython system-wide
COPY ipython_kernel_config.py "/etc/ipython/"
RUN fix-permissions "/etc/ipython/"

ENV PYARROW_IGNORE_TIMEZONE=1

USER ${NB_UID}

# Install pyarrow
RUN mamba install --quiet --yes \
    'pyarrow' && \
    mamba clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

WORKDIR "${HOME}"
