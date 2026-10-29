ARG SIGNALK_VERSION=latest
FROM signalk/signalk-server:${SIGNALK_VERSION}
USER root
RUN apt-get update && apt-get install -y --no-install-recommends  \
    alsa-utils \
    mpg321 \
    festival \
    festvox-kallpc16k \
    && rm -rf /var/lib/apt/lists/*
USER node
