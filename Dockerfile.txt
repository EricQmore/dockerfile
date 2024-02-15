ARG NODE_IMAGE

FROM ${NODE_IMAGE} AS build

# Install build dependencies
RUN apt-get update && apt-get install -y make perl

WORKDIR /

COPY package.json package-lock.json ./
COPY src/shadowbox/package.json src/shadowbox/
RUN --mount=type=cache,target=/root/.npm npm ci

COPY scripts scripts/
COPY src src/
COPY tsconfig.json ./
COPY third_party third_party

ARG ARCH
ARG VERSION

RUN ARCH=${ARCH} ROOT_DIR=/ SB_VERSION=${VERSION} npm run action shadowbox/server/build

FROM ${NODE_IMAGE}

LABEL shadowbox.node_version=16.18.0
LABEL shadowbox.github.release=${VERSION}

STOPSIGNAL SIGKILL

# Install runtime dependencies
RUN apt-get update && apt-get install -y coreutils curl

COPY src/shadowbox/scripts scripts/
COPY src/shadowbox/scripts/update_mmdb.sh /etc/periodic/weekly/update_mmdb

RUN /etc/periodic/weekly/update_mmdb

RUN mkdir -p /root/shadowbox/persisted-state

WORKDIR /opt/outline-server
COPY --from=build /build/shadowbox/ .

COPY src/shadowbox/docker/cmd.sh /
CMD /cmd.sh
