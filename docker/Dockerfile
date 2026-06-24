# Copyright (c) Meta Platforms, Inc. and affiliates.

FROM micr.cloud.mioffice.cn/cockpit-build/ubuntu2204:v0.4.8

USER root

COPY jammy-install-deps.sh /
COPY download-ndk.sh /

RUN bash -c "/jammy-install-deps.sh" && \
    bash -c "/download-ndk.sh /opt/ndk"

WORKDIR /ExtendedAndroidTools

ENTRYPOINT ["/bin/bash"]
