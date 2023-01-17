# Basic virtualbox host in archlinux container

FROM archlinux/archlinux

ARG VMDIR
ARG VBUSR
ARG VBUID
ARG TZONE

ENV VMDIR=$VMDIR
ENV VBUID=$VBUID

RUN set -xe -o pipefail && \
    pacman -Sy --noconfirm --needed && \
    pacman -S  --noconfirm --needed \
        virtualbox-host-modules-arch \
        virtualbox-guest-iso \
        virtualbox \
        && \
    pacman -Scc --noconfirm && \
    rm -f /var/cache/pacman/pkg/* && \
    rm -f /usr/src/*

COPY "dist/bin/"  "/usr/local/bin/"

RUN set -xe -o pipefail && \
    chmod -cR u=rwx,go=rx "/usr/local/bin" && \
    VBUSR=$VBUSR TZONE=$TZONE /usr/local/bin/setup && \
    rm /usr/local/bin/setup

VOLUME ["$VMDIR", "/home/$VBUSR"]

WORKDIR "/home/$VBUSR"

USER $VBUSR:$VBUSR

HEALTHCHECK \
   --interval=60s \
   --timeout=15s \
   --start-period=30s \
   CMD ["/usr/local/bin/entrypoint", "healthcheck"]

ENTRYPOINT ["/usr/local/bin/entrypoint"]
CMD []
