################################################################################
# Beginning of multi stage Dockerfile
################################################################################
ARG BASE_IMAGE=redhat/ubi8-minimal:8.4-208
#ARG BASE_IMAGE=centos:8.3.2011
################################################################################
# Build stage 0 `builder`:
# Extract OpenSearch artifact
################################################################################
FROM ${BASE_IMAGE} AS builder
#
ARG VERSION=1.0.1
ARG SOURCE_OPENSEARCH=https://artifacts.opensearch.org/releases/bundle/opensearch/${VERSION}/opensearch-${VERSION}-linux-x64.tar.gz

# `tini` is a tiny but valid init for containers. This is used to cleanly
# control how OpenSearch and any child processes are shut down.
#
# The tini GitHub page gives instructions for verifying the binary using
# gpg, but the keyservers are slow to return the key and this can fail the
# build. Instead, we check the binary against the published checksum.
RUN set -eux ; \
    tini_bin="" ; \
    case "$(arch)" in \
        aarch64) tini_bin='tini-arm64' ;; \
        x86_64)  tini_bin='tini-amd64' ;; \
        *) echo >&2 ; echo >&2 "Unsupported architecture $(arch)" ; echo >&2 ; exit 1 ;; \
    esac ; \
    curl --retry 8 -S -L -O https://github.com/krallin/tini/releases/download/v0.19.0/${tini_bin} ; \
    curl --retry 8 -S -L -O https://github.com/krallin/tini/releases/download/v0.19.0/${tini_bin}.sha256sum ; \
    sha256sum -c ${tini_bin}.sha256sum ; \
    rm ${tini_bin}.sha256sum ; \
    mv ${tini_bin} /tini ; \
    chmod +x /tini
RUN mkdir /usr/share/opensearch
WORKDIR /usr/share/opensearch
# OpenSearch
ARG PACKAGE_MANAGER=microdnf

RUN for iter in {1..10}; do \
      ${PACKAGE_MANAGER} update --disableplugin=subscription-manager --setopt=tsflags=nodocs -y && \
      ${PACKAGE_MANAGER} install --disableplugin=subscription-manager --setopt=tsflags=nodocs -y \
        tar gzip zip unzip && \
      ${PACKAGE_MANAGER} --disableplugin=subscription-manager clean all && exit_code=0 && break || exit_code=$? && echo "${PACKAGE_MANAGER} error: retry $iter in 10s" && \
      sleep 10; \
    done; \
    (exit $exit_code)
RUN curl --retry 8 -S -L -O ${SOURCE_OPENSEARCH} && \
    tar zxf opensearch-${VERSION}-linux-x64.tar.gz --strip-components=1 && \
    rm -f opensearch-${VERSION}-linux-x64.tar.gz

RUN sed -i -e 's/OPENSEARCH_DISTRIBUTION_TYPE=tar/OPENSEARCH_DISTRIBUTION_TYPE=docker/' /usr/share/opensearch/bin/opensearch-env
RUN mkdir -p config config/jvm.options.d data logs
RUN chmod 0775 config config/jvm.options.d data logs
COPY config/opensearch.yml config/log4j2.properties config/
RUN chmod 0660 config/opensearch.yml config/log4j2.properties
################################################################################
# Build stage 1 (the actual OpenSearch image):
#
# Copy opensearch from stage 0
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
        nc shadow-utils zip unzip findutils && \
      ${PACKAGE_MANAGER} --disableplugin=subscription-manager clean all && exit_code=0 && break || exit_code=$? && echo "${PACKAGE_MANAGER} error: retry $iter in 10s" && \
      sleep 10; \
    done; \
    (exit $exit_code)
RUN groupadd -g 1000 opensearch && \
    adduser -u 1000 -g 1000 -G 0 -d /usr/share/opensearch opensearch && \
    chmod 0775 /usr/share/opensearch && \
    chown -R 1000:0 /usr/share/opensearch
WORKDIR /usr/share/opensearch
COPY --from=builder --chown=1000:0 /usr/share/opensearch /usr/share/opensearch
COPY --from=builder --chown=0:0 /tini /tini
# Replace OpenJDK's built-in CA certificate keystore with the one from the OS
# vendor. The latter is superior in several ways.
# REF: https://github.com/elastic/elasticsearch-docker/issues/171
RUN ln -sf /etc/pki/ca-trust/extracted/java/cacerts /usr/share/opensearch/jdk/lib/security/cacerts

ENV PATH /usr/share/opensearch/bin:$PATH
ENV JAVA_HOME /usr/share/opensearch/jdk

COPY bin/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh

# The JDK's directories' permissions don't allow `java` to be executed under a different
# group to the default. Fix this.
RUN find /usr/share/opensearch/jdk -type d -exec chmod 0755 '{}' \; && \
    chmod g=u /etc/passwd && \
    chmod 0775 /usr/local/bin/docker-entrypoint.sh

# Ensure that there are no files with setuid or setgid, in order to mitigate "stackclash" attacks.
RUN find / -xdev -perm -4000 -exec chmod ug-s {} +

EXPOSE 9200 9300

LABEL org.label-schema.license="${LICENSE}" \
  org.label-schema.name="OpenSearch" \
  org.label-schema.schema-version="1.0" \
  org.label-schema.url="https://www.opensearch.org" \
  org.label-schema.usage="https://www.opensearch.org/guide/en/opensearch/reference/index.html" \
  org.label-schema.vcs-url="https://github.com/opensearch-project/OpenSearch" \
  org.label-schema.vendor="OpenSearch" \
  org.label-schema.version="${VERSION}" \
  org.opencontainers.image.documentation="https://www.opensearch.org/guide/en/opensearch/reference/index.html" \
  org.opencontainers.image.licenses="${LICENSE}" \
  org.opencontainers.image.source="https://github.com/opensearch-project/OpenSearch" \
  org.opencontainers.image.title="OpenSearch" \
  org.opencontainers.image.url="https://www.opensearch.org" \
  org.opencontainers.image.vendor="OpenSearch" \
  org.opencontainers.image.version="${VERSION}"

ENTRYPOINT ["/tini", "--", "/usr/local/bin/docker-entrypoint.sh"]
# Dummy overridable parameter parsed by entrypoint
CMD ["opensearchwrapper"]

################################################################################
# End of multi-stage Dockerfile
################################################################################