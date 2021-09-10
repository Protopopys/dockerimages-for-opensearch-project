################################################################################
# Beginning of multi stage Dockerfile
################################################################################
ARG BASE_IMAGE=redhat/ubi8-minimal:8.4-208
#ARG BASE_IMAGE=centos:8.3.2011
################################################################################
# Build stage 0 `builder`:
# Extract OpenSearch Dashboards artifact
################################################################################
FROM ${BASE_IMAGE} AS builder
#
ARG VERSION=1.0.1
ARG SOURCE_OPENSEARCH_DASHBOARDS=https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/${VERSION}/opensearch-dashboards-${VERSION}-linux-x64.tar.gz

RUN mkdir /usr/share/opensearch-dashboards
WORKDIR /usr/share/opensearch-dashboards
# OpenSearch Dashboards
ARG PACKAGE_MANAGER=microdnf

RUN for iter in {1..10}; do \
      ${PACKAGE_MANAGER} update --disableplugin=subscription-manager --setopt=tsflags=nodocs -y && \
      ${PACKAGE_MANAGER} install --disableplugin=subscription-manager --setopt=tsflags=nodocs -y \
        tar gzip findutils && \
      ${PACKAGE_MANAGER} --disableplugin=subscription-manager clean all && exit_code=0 && break || exit_code=$? && echo "${PACKAGE_MANAGER} error: retry $iter in 10s" && \
      sleep 10; \
    done; \
    (exit $exit_code)
RUN curl --retry 8 -S -L -O ${SOURCE_OPENSEARCH_DASHBOARDS} && \
    tar zxf opensearch-dashboards-${VERSION}-linux-x64.tar.gz --strip-components=1 && \
    rm -f opensearch-dashboards-${VERSION}-linux-x64.tar.gz

# Set some OpenSearch Dashboards configuration defaults.
COPY config/opensearch_dashboards.yml config/opensearch_dashboards.yml
RUN chmod 0660 config/opensearch_dashboards.yml
# Ensure that group permissions are the same as user permissions.
# This will help when relying on GID-0 to run OpenSearch Dashboards, rather than UID-1000.
# OpenShift does this, for example.
# REF: https://docs.openshift.org/latest/creating_images/guidelines.html
RUN chmod -vR g=u /usr/share/opensearch-dashboards
RUN find /usr/share/opensearch-dashboards -type d -exec chmod g+s {} \;
################################################################################
# Build stage 1 (the actual OpenSearch Dashboards image):
#
# Copy opensearch-dashboards from stage 0
# Add entrypoint
################################################################################
FROM ${BASE_IMAGE}
ENV OPENSEARCH_CONTAINER true
ARG VERSION=1.0.1
ARG LICENSE="Apache-2.0 License"
#
ARG PACKAGE_MANAGER=microdnf
RUN for iter in {1..10}; do \
      ${PACKAGE_MANAGER} update --disableplugin=subscription-manager --setopt=tsflags=nodocs -y && \
      ${PACKAGE_MANAGER} install --disableplugin=subscription-manager --setopt=tsflags=nodocs -y \
        fontconfig freetype shadow-utils libnss3.so findutils && \
      ${PACKAGE_MANAGER} --disableplugin=subscription-manager clean all && exit_code=0 && break || exit_code=$? && echo "${PACKAGE_MANAGER} error: retry $iter in 10s" && \
      sleep 10; \
    done; \
    (exit $exit_code)

# Add an init process, check the checksum to make sure it's a match
RUN curl -L -o /usr/local/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v1.2.2/dumb-init_1.2.2_amd64
RUN echo "37f2c1f0372a45554f1b89924fbb134fc24c3756efaedf11e07f599494e0eff9  /usr/local/bin/dumb-init" | sha256sum -c -
RUN chmod +x /usr/local/bin/dumb-init

RUN mkdir /usr/share/fonts/local
RUN curl -L -o /usr/share/fonts/local/NotoSansCJK-Regular.ttc https://github.com/googlefonts/noto-cjk/raw/NotoSansV2.001/NotoSansCJK-Regular.ttc
RUN echo "5dcd1c336cc9344cb77c03a0cd8982ca8a7dc97d620fd6c9c434e02dcb1ceeb3  /usr/share/fonts/local/NotoSansCJK-Regular.ttc" | sha256sum -c -
RUN fc-cache -v

# Provide a non-root user to run the process.
RUN groupadd -g 1000 opensearch-dashboards && \
    adduser -u 1000 -g 1000 -G 0 -d /usr/share/opensearch-dashboards opensearch-dashboards && \
    chmod 0775 /usr/share/opensearch-dashboards && \
    chown -R 1000:0 /usr/share/opensearch-dashboards && \
    find /usr/share/opensearch-dashboards -type d -exec chmod g+s {} \;
WORKDIR /usr/share/opensearch-dashboards
# Bring in OpenSearch Dashboards from the initial stage.
COPY --from=builder --chown=1000:0 /usr/share/opensearch-dashboards /usr/share/opensearch-dashboards

ENV PATH=/usr/share/opensearch-dashboards/bin:$PATH

# Add the launcher/wrapper script. It knows how to interpret environment
# variables and translate them to OpenSearch Dashboards CLI options.
COPY --chown=1000:0 bin/opensearch-dashboards-docker /usr/local/bin/

# Ensure gid 0 write permissions for OpenShift.
RUN chmod -v g+ws /usr/share/opensearch-dashboards && \
    find /usr/share/opensearch-dashboards -gid 0 -and -not -perm /g+w -exec chmod g+w {} \;

# Ensure that there are no files with setuid or setgid, in order to mitigate "stackclash" attacks.
RUN find / -xdev -perm -4000 -exec chmod ug-s {} +

LABEL org.label-schema.license="${LICENSE}" \
  org.label-schema.name="OpenSearch Dashboards" \
  org.label-schema.schema-version="1.0" \
  org.label-schema.url="https://www.opensearch.org" \
  org.label-schema.usage="https://www.opensearch.org/guide/en/opensearch/reference/index.html" \
  org.label-schema.vcs-url="https://github.com/opensearch-project/OpenSearch-Dashboards" \
  org.label-schema.vendor="OpenSearch" \
  org.label-schema.version="${VERSION}" \
  org.opencontainers.image.documentation="https://www.opensearch.org/docs" \
  org.opencontainers.image.licenses="${LICENSE}" \
  org.opencontainers.image.source="https://github.com/opensearch-project/OpenSearch-Dashboards" \
  org.opencontainers.image.title="OpenSearch Dashboards" \
  org.opencontainers.image.url="https://www.opensearch.org" \
  org.opencontainers.image.vendor="OpenSearch" \
  org.opencontainers.image.version="${VERSION}"

USER opensearch-dashboards

ENTRYPOINT ["/usr/local/bin/dumb-init", "--"]

CMD ["/usr/local/bin/opensearch-dashboards-docker"]