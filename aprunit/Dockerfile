FROM            hwdm/apbase:latest
MAINTAINER      Howard Mei      <howardleomei@gmail.com>

ARG             V=2.1.2-r3
#ARG             GODNSMASQ_VERSION=1.0.7
ENV             RUNSVWAITSEC=1

## go-dnsmasq is unnecessary for alpine >= v3.4
#ADD             https://github.com/janeczku/go-dnsmasq/releases/download/${GODNSMASQ_VERSION}/go-dnsmasq-min_linux-amd64   /bin/go-dnsmasq

COPY                etc         /etc
COPY                usr         /usr
COPY                entrypoint  /entrypoint


RUN             apk-install libressl ca-certificates bind-tools curl runit=${V} && \
                #chmod a+x /bin/go-dnsmasq && chmod a+x /etc/preinit.d/* && \
                #addgroup go-dnsmasq && adduser -D -g "GO-DNSMASQ Daemon" -s /bin/sh -G go-dnsmasq go-dnsmasq && \
                #setcap CAP_NET_BIND_SERVICE=+eip /bin/go-dnsmasq && \
                #chown root:go-dnsmasq /etc/resolv.conf && chmod 0664 /etc/resolv.conf && \
                apk-cleanup


EXPOSE          53  53/udp

# Define the Entry Point and/or Default Command
ENTRYPOINT      ["/entrypoint"]
