ARG BUILD_FROM
FROM $BUILD_FROM

ARG EUFY_SECURITY_WS_VERSION
ARG EUFY_SECURITY_CLIENT_COMMIT

COPY patches /tmp/eufy-c30-patches

WORKDIR /tmp/build-client

RUN \
    set -eux \
    && apk add --no-cache \
        git \
        jq \
        nodejs \
        npm \
    && git clone https://github.com/bropat/eufy-security-client.git . \
    && git fetch origin refs/pull/797/head \
    && git checkout "${EUFY_SECURITY_CLIENT_COMMIT}" \
    && git apply /tmp/eufy-c30-patches/*.patch \
    && npm install --ignore-scripts \
    && npm install --ignore-scripts --no-save copyfiles@2.4.1 \
    && npm run build \
    && npm pack \
    && mv eufy-security-client-*.tgz /tmp/eufy-security-client-c30-mqtt.tgz

WORKDIR /usr/src/app

RUN \
    set -eux \
    && npm install --force \
        "eufy-security-ws@${EUFY_SECURITY_WS_VERSION}" \
    && npm install --force \
        /tmp/eufy-security-client-c30-mqtt.tgz \
    && rm -rf \
        node_modules/eufy-security-ws/node_modules/eufy-security-client \
    && rm -f \
        /tmp/eufy-security-client-c30-mqtt.tgz \
    && rm -rf \
        /tmp/build-client \
        /tmp/eufy-c30-patches

COPY run.sh /
RUN chmod a+x /run.sh

WORKDIR /data
CMD ["/run.sh"]
