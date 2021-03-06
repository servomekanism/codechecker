###############################################################################
#-----------------------------    BUILD STAGE   ------------------------------#
###############################################################################

FROM python:2.7.16-slim-stretch as builder

ARG CC_VERSION=master
ENV CC_VERSION ${CC_VERSION}

ARG INSTALL_AUTH=no
ARG INSTALL_PG8000=no
ARG INSTALL_PSYCOPG2=no

ARG DEBIAN_FRONTEND=noninteractive
RUN set -x && apt-get update -qq \
  && apt-get install -qqy --no-install-recommends \
    ca-certificates \
    curl \
    doxygen \
    git \
    thrift-compiler \
    make

RUN if [ "$INSTALL_AUTH" = "yes" ] ; then \
      apt-get install -qqy --no-install-recommends \
        libldap2-dev \
        libsasl2-dev \
        libssl-dev; \
    fi

# Download CodeChecker release.
RUN git clone https://github.com/Ericsson/CodeChecker.git /codechecker
WORKDIR /codechecker
RUN git checkout ${CC_VERSION}

# Build CodeChecker web.
RUN make -C /codechecker/web package

# Remove dojo uncompressed files.
WORKDIR /codechecker/web/build/CodeChecker
RUN find www/scripts/plugins/dojo/ -name *.uncompressed.js -exec rm -rf '{}' \;

# Copy necessary runtime environment files to the build directory.
RUN mkdir -p /codechecker/web/build/CodeChecker/requirements_py && \
  cp /codechecker/web/requirements.txt /codechecker/web/build/CodeChecker/requirements_py \
  && if [ "$INSTALL_PG8000" = "yes" ] ; then \
       cp /codechecker/web/requirements_py/db_pg8000/requirements.txt \
          /codechecker/web/build/CodeChecker/requirements_py/db_pg8000_requirements.txt; \
     fi \
  && if [ "$INSTALL_PSYCOPG2" = "yes" ] ; then \
       cp /codechecker/web/requirements_py/db_psycopg2/requirements.txt \
          /codechecker/web/build/CodeChecker/requirements_py/db_psycopg2_requirements.txt; \
     fi \
  && if [ "$INSTALL_AUTH" = "yes" ] ; then \
       cp /codechecker/web/requirements_py/auth/requirements.txt \
       /codechecker/web/build/CodeChecker/requirements_py/auth.txt_requirements; \
     fi

###############################################################################
#--------------------------    PRODUCTION STAGE   ----------------------------#
###############################################################################

FROM python:2.7.16-slim-stretch

ARG CC_GID=950
ARG CC_UID=950

ENV CC_GID ${CC_GID}
ENV CC_UID ${CC_UID}

ENV TINI_VERSION v0.18.0

RUN set -x && apt-get update -qq \
  # Prevent fail when install postgresql-client.
  && mkdir -p /usr/share/man/man1 \
  && mkdir -p /usr/share/man/man7 \
  \
  && apt-get install -qqy --no-install-recommends ca-certificates \
    postgresql-client \
    # To switch user and exec command.
    gosu

COPY --from=builder /codechecker/web/build/CodeChecker /codechecker/

# Install python requirements.
RUN apt-get install -qqy --no-install-recommends \
  python-dev \
  # gcc is needed to build psutil.
  gcc \
  \
  && for filename in /codechecker/requirements_py/*.txt; do \
       pip install -r $filename; \
     done \
  \
  # Remove unnecessary packages.
  && pip uninstall -y wheel \
  && apt-get purge -y --auto-remove \
    gcc \
    python-dev \
  \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/ \
  && set +x

# Create user and group for CodeChecker.
RUN groupadd -r codechecker -g ${CC_GID} \
  && useradd -r --no-log-init -M -u ${CC_UID} -g codechecker codechecker

# Change permission of the CodeChecker package.
RUN chown codechecker:codechecker /codechecker

ENV PATH="/codechecker/bin:$PATH"

COPY ./entrypoint.sh /usr/local/bin/
RUN chmod a+x /usr/local/bin/entrypoint.sh \
  && chown codechecker:codechecker /usr/local/bin/entrypoint.sh

ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini

EXPOSE 8001

ENTRYPOINT ["/tini", "--", "/usr/local/bin/entrypoint.sh"]

CMD ["CodeChecker", "server", "--workspace", "/workspace", "--not-host-only"]
