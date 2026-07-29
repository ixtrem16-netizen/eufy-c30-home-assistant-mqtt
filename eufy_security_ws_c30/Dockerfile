ARG BUILD_FROM
FROM $BUILD_FROM

ARG EUFY_SECURITY_WS_VERSION
ARG EUFY_SECURITY_CLIENT_COMMIT

WORKDIR /tmp/build-client

RUN \
    set -eux \
    && apk add --no-cache \
        git \
        jq \
        nodejs \
        npm \
    && git clone https://github.com/bropat/eufy-security-client.git . \
    && git checkout "${EUFY_SECURITY_CLIENT_COMMIT}" \
    && npm install --ignore-scripts \
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
        /tmp/build-client

COPY run.sh /
RUN chmod a+x /run.sh

WORKDIR /data
CMD ["/run.sh"]
