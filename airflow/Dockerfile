# syntax=docker/dockerfile:1
ARG GIT_SYNC
ARG STATSD_EXPORTER

FROM oci.stackable.tech/mirror/prom/statsd-exporter:${STATSD_EXPORTER} as statsd-exporter

# For updated versions check https://github.com/kubernetes/git-sync/releases
# which should contain a image location (e.g. registry.k8s.io/git-sync/git-sync:v3.6.8)
FROM oci.stackable.tech/mirror/git-sync/git-sync:${GIT_SYNC} as gitsync-image


FROM stackable/image/vector AS airflow-build-image

ARG PRODUCT
ARG PYTHON

COPY airflow/constraints-${PRODUCT}-python${PYTHON}.txt /tmp/constraints.txt

# the mysql provider is currently excluded.
# Requires implementation of https://github.com/apache/airflow/blob/2.2.5/scripts/docker/install_mysql.sh
ENV AIRFLOW_EXTRAS=async,amazon,celery,cncf.kubernetes,docker,dask,elasticsearch,ftp,grpc,hashicorp,http,ldap,google,google_auth,microsoft.azure,odbc,pandas,postgres,redis,sendgrid,sftp,slack,ssh,statsd,virtualenv,trino

RUN microdnf update && \
    microdnf install \
        cyrus-sasl-devel \
        gcc \
        gcc-c++ \
        libpq-devel \
        openldap-devel \
        openssl-devel \
        python${PYTHON}-devel \
        python${PYTHON}-pip \
        python${PYTHON}-wheel \
        unixODBC-devel && \
    microdnf clean all && \
    python3 -m venv --system-site-packages /stackable/app && \
    source /stackable/app/bin/activate && \
    pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir apache-airflow[${AIRFLOW_EXTRAS}]==${PRODUCT} --constraint /tmp/constraints.txt && \
    # Needed for pandas S3 integration to e.g. write and read csv and parquet files to/from S3
    pip install --no-cache-dir s3fs


FROM stackable/image/vector AS airflow-main-image

ARG PRODUCT
ARG PYTHON
ARG RELEASE
ARG TINI

LABEL name="Apache Airflow" \
      maintainer="info@stackable.tech" \
      vendor="Stackable GmbH" \
      version="${PRODUCT}" \
      release="${RELEASE}" \
      summary="The Stackable image for Apache Airflow." \
      description="This image is deployed by the Stackable Operator for Apache Airflow."

COPY airflow/licenses /licenses

# Update image and install python
RUN microdnf update && \
    microdnf install  \
    ca-certificates \
    cyrus-sasl \
    git \
    libpq \
    openldap \
    openldap-clients \
    openssh-clients \
    openssl-libs \
    openssl-pkcs11 \
    python${PYTHON} \
    socat \
    unixODBC && \
    microdnf clean all && \
    rm -rf /var/cache/yum

ENV HOME=/stackable
ENV AIRFLOW_USER_HOME_DIR=/stackable
ENV PATH=$PATH:/bin:$HOME/app/bin
ENV AIRFLOW_HOME=$HOME/airflow

RUN mkdir -pv ${AIRFLOW_HOME} && \
    mkdir -pv ${AIRFLOW_HOME}/dags && \
    mkdir -pv ${AIRFLOW_HOME}/logs

RUN chown --recursive stackable:stackable ${AIRFLOW_HOME}

# Get the correct `tini` binary for our architecture.
# It is used as an init alternative in the entrypoint
RUN curl --fail -o /usr/bin/tini "https://repo.stackable.tech/repository/packages/tini/tini-${TINI}-$(arch)"
COPY airflow/stackable/utils/entrypoint.sh /entrypoint.sh
COPY airflow/stackable/utils/run-airflow.sh /run-airflow.sh
RUN chmod a+x /entrypoint.sh && \
    chmod a+x /run-airflow.sh && \
    chmod +x /usr/bin/tini

COPY --from=statsd-exporter --chown=stackable:stackable /bin/statsd_exporter /stackable/statsd_exporter
COPY --from=airflow-build-image --chown=stackable:stackable /stackable/ ${HOME}/
COPY --from=gitsync-image --chown=stackable:stackable /git-sync /stackable/git-sync

USER stackable
WORKDIR /stackable

ENTRYPOINT ["/usr/bin/tini", "--", "/run-airflow.sh"]
CMD []
