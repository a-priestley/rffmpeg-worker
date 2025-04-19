ARG VERSION
FROM public.ecr.aws/docker/library/alpine:3.18 as builder
ARG VERSION
ARG TARGETOS
ARG TARGETARCH
ARG TARGETVARIANT=""
ARG TARGETPLATFORM
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=${TARGETOS} \
    GOARCH=${TARGETARCH} \
    GOARM=${TARGETVARIANT} \
    GOPATH=/go
ENV PATH $GOPATH/bin:$PATH
#hadolint ignore=DL3018
RUN \
    apk add --no-cache go upx \
    && go install -ldflags="-s -w" github.com/drone/envsubst/cmd/envsubst@latest \
    && upx /go/bin/envsubst

FROM public.ecr.aws/docker/library/ubuntu:jammy-20221101

ARG VERSION
ARG CHANNEL
ARG TARGETPLATFORM
ENV TARGETPLATFORM=${TARGETPLATFORM:-linux/amd64}

ENV \
    DEBCONF_NONINTERACTIVE_SEEN=true \
    DEBIAN_FRONTEND="noninteractive" \
    APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=DontWarn \
    UMASK="0002" \
    TZ="Etc/UTC"

ENV NVIDIA_DRIVER_CAPABILITIES="compute,video,utility"

USER root

WORKDIR /app

RUN \
    set -eux \
    && echo 'APT::Install-Recommends "false";' >/etc/apt/apt.conf.d/00recommends \
    && echo 'APT::Install-Suggests "false";' >>/etc/apt/apt.conf.d/00recommends \
    && echo 'APT::Get::Install-Recommends "false";' >>/etc/apt/apt.conf.d/00recommends \
    && echo 'APT::Get::Install-Suggests "false";' >>/etc/apt/apt.conf.d/00recommends \
    && \
    apt-get -qq update \
    && \
    apt-get install -y \
        bash \
        ca-certificates \
        curl \
        dnsutils \
        gnupg2 \
        iputils-ping \
        jo \
        jq \
        locales \
        moreutils \
        pv \
        software-properties-common \
        tini \
        tzdata \
        vim-tiny \
        unrar \
        unzip \
        wget \
    && \
    localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8 \
    && ln -s /usr/bin/vim.tiny /usr/local/bin/vi \
    && ln -s /usr/bin/vim.tiny /usr/local/bin/vim \
    && ln -s /usr/bin/vim.tiny /usr/local/bin/nano \
    && ln -s /usr/bin/vim.tiny /usr/local/bin/emacs \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && apt-get autoremove -y \
    && apt-get clean \
    && \
    rm -rf \
        /tmp/* \
        /var/lib/apt/lists/* \
        /var/cache/apt/* \
        /var/tmp/*

ENV LANG en_US.UTF-8

RUN \
    apt-get -qq update \
    && \
    add-apt-repository universe \
    && \
    case "${TARGETPLATFORM}" in \
        'linux/amd64') \
            curl -sL https://repositories.intel.com/graphics/intel-graphics.key | apt-key add - \
            && echo 'deb [arch=amd64] https://repositories.intel.com/graphics/ubuntu jammy main' > /etc/apt/sources.list.d/intel.list \
            && \
            export ARCH='amd64'; \
            export EXTRA_INSTALL_ARG='intel-media-va-driver-non-free nvidia-opencl-icd-340 i965-va-driver mesa-va-drivers intel-opencl-icd'; \
        ;; \
        'linux/arm64') \
            apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 6587FFD6536B8826E88A62547876AE518CBCF2F2 \
            && echo "deb http://ppa.launchpad.net/ubuntu-raspi2/ppa-nightly/ubuntu focal main" | tee /etc/apt/sources.list.d/raspberrypi.list \
            && \
            export ARCH='arm64'; \
            export EXTRA_INSTALL_ARG='libomxil-bellagio0 libomxil-bellagio-bin libraspberrypi0'; \
            unset NVIDIA_DRIVER_CAPABILITIES; \
        ;; \
    esac \
    && \
    mkdir -p /etc/apt/keyrings && \
    curl -fsSL https://repo.jellyfin.org/jellyfin_team.gpg.key | apt-key add - && \
    echo "deb [arch=${ARCH}] https://repo.jellyfin.org/ubuntu jammy main" | tee /etc/apt/sources.list.d/jellyfin.list && \
    apt-get -qq update \
    && \
    apt-get -qq install -y \
        ${EXTRA_INSTALL_ARG} \
    && \
    apt-get -qq update && \
    apt-get -qq install -y \
        jellyfin-ffmpeg7 \
        openssh-server \
        augeas-tools \
        libvshadow-utils \
    && \
    apt-get remove -y gnupg2 \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && apt-get autoremove -y \
    && apt-get clean \
    && \
    rm -rf \
        /tmp/* \
        /var/lib/apt/lists/* \
        /var/tmp/ \
    && printf "umask %d\n" "${UMASK}" >> /etc/bash.bashrc

RUN mkdir -p /etc/authorized_keys \
    && chmod -R 700 /etc/authorized_keys \
    && echo "AuthorizedKeysFile .ssh/authorized_keys /etc/authorized_keys/%u" | tee -a /etc/ssh/sshd_config \
    && cp -a /etc/ssh /etc/ssh.cache

EXPOSE 22

COPY --from=builder /go/bin/envsubst /usr/local/bin/envsubst
COPY ./docker/entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["/usr/sbin/sshd", "-D", "-e", "-f", "/etc/ssh/sshd_config"]

LABEL org.opencontainers.image.source="https://github.com/nouun/rffmpeg-worker"
