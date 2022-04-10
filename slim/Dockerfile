# Volatility, on Docker 🐳
#
#--------------------------
# Staging Container
#--------------------------

ARG IMAGE_NAME="python"
ARG CONTAINER_TAG="3.10-alpine"

FROM "${IMAGE_NAME}:${CONTAINER_TAG}" AS stage

ARG GIT_TAG_PYTHON_YARA="master"
ARG GIT_TAG_VOLATILITY3="stable"

# The virtual del is not done in the same RUN command so size will not be decreased
RUN apk add --no-cache --virtual=stage \
        curl \
        git \
        unzip

RUN python3 -m ensurepip --default-pip

RUN cd /tmp                                             && \
    mkdir -p /stage                                     && \
    # Build and install Volatility3
    git clone --branch="${GIT_TAG_VOLATILITY3}" --depth=1 --single-branch \
        https://github.com/volatilityfoundation/volatility3.git       && \
    cd volatility3                                                    && \
    python3 setup.py install --root /stage                            && \
    # Make PDBConv able to be ran without explicitly invoking python3
    find /stage -name "pdbconv.py" -exec sed -i '1i #!/usr/local/bin/python3' {} \;

# The virtual del is not done in the same RUN command so size will not be decreased
RUN apk --purge del \
        stage

#--------------------------
# Production Container
#--------------------------

FROM "${IMAGE_NAME}:${CONTAINER_TAG}" AS production

ARG INSTALL_GROUP="ci"
ARG INSTALL_USER="unprivileged"
ARG INSTALL_PREFIX="/usr/local/lib/python3.10/site-packages"

ENV LANG="C.UTF-8"
ENV LC_ALL="C.UTF-8"

ENV PYTHONFAULTHANDLER="1"
ENV PYTHONHASHSEED="random"
ENV PYTHONUNBUFFERED="1"

# Create a non-root user
RUN addgroup -S "${INSTALL_GROUP}" && \
    adduser -DG -S "${INSTALL_USER}" -G "${INSTALL_GROUP}" -g "Unprivileged user,,,,"

# Install needed packages and temporary ones in a virtual package
# In order to decrease overall image size the virtual package
# Creation and Deletion *MUST* be in the same RUN command
RUN apk add --no-cache \
        dumb-init \
    && \
    apk add --no-cache --virtual=stage \
        gcc \
        musl-dev \
    && \ 
    python3 -m ensurepip --default-pip && \
    python3 -m pip install --no-cache --upgrade \
        wheel && \
    python3 -m pip install --no-cache --upgrade \
        pefile \
    && \
    apk --purge del \
        stage

# Copy Built Volatility3, and PDBParse files
COPY --from=stage --chown="${INSTALL_USER}:${INSTALL_GROUP}" \
        /stage /

# Setup Vol3 and pdbconv (symlinks, permissions)
# Create final workdir and adjust permissions
RUN cd "${INSTALL_PREFIX}"/volatility3 && \
    chmod 0755 \
        framework/symbols/windows/pdbconv.py && \
    for destination in "v3" "vol.py" "vol3" "volatility" "volatility3"; \
    do ln -sf \
        "/usr/local/bin/vol" \
        "/usr/local/bin/${destination}"; \
    done && \
    ln -sf \
        "${INSTALL_PREFIX}/volatility3/framework/symbols/windows/pdbconv.py" \
        "/usr/local/bin/pdbconv" && \
    ln -sf \
        "${INSTALL_PREFIX}/volatility3/framework/symbols/windows/pdbconv.py" \
        "/usr/local/bin/pdbconv.py" && \
    chown -R "${INSTALL_USER}:${INSTALL_GROUP}" /usr/local/bin  && \
    mkdir -p /workdir                                           && \
    chown -R "${INSTALL_USER}:${INSTALL_GROUP}" /workdir

# Setup Symbols (directories, symlinks, permissions)
RUN mkdir -p /symbols                                       && \
    mkdir /symbols/mac                                      && \
    mkdir /symbols/linux                                    && \
    mkdir /symbols/windows                                  && \
    ln -sf /symbols/mac \
       "${INSTALL_PREFIX}"/volatility3/symbols  && \
    ln -sf /symbols/linux \
       "${INSTALL_PREFIX}"/volatility3/symbols  && \
    ln -sf /symbols/windows \
      "${INSTALL_PREFIX}"/volatility3/symbols   && \
    chown -R "${INSTALL_USER}:${INSTALL_GROUP}" /symbols && \
    chown -R "${INSTALL_USER}:${INSTALL_GROUP}" "${INSTALL_PREFIX}"/volatility3/symbols && \
    cd "${INSTALL_PREFIX}"/volatility3/symbols  && \
    find . -type d -exec chmod 0777 "{}" \; && \
    find . -type f -exec chmod 0666 "{}" \; && \
    find /usr/local/lib/python3* -type d -name symbols -exec chmod 0777 "{}" \;

VOLUME [ "/symbols" ]

ENV PATH="/usr/local/bin:${PATH}"

WORKDIR "/workdir"

USER "${INSTALL_USER}"

ENTRYPOINT [ "/usr/bin/dumb-init", "--", "volatility3" ]

CMD [ "--help" ]