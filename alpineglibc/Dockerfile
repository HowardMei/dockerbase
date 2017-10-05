FROM            hwdm/apbase:latest
MAINTAINER      Howard Mei      <howardleomei@gmail.com>

ENV             LANG=en_US.UTF-8
ENV             LANGUAGE=en_US:en
ENV             LC_ALL=en_US.UTF-8
ENV             GLIBC_VERSION=2.25-r1

# Set Default User and Home Dir
USER            root

# Here we install GNU libc (aka glibc) and set C.UTF-8 locale as default.
RUN             apk add --no-cache --virtual=.build-dependencies wget ca-certificates && \
                wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/${GLIBC_VERSION}/sgerrand.rsa.pub -O /etc/apk/keys/sgerrand.rsa.pub && \
                for pkg in glibc-${GLIBC_VERSION} glibc-bin-${GLIBC_VERSION} glibc-i18n-${GLIBC_VERSION}; do \
                    wget -cL --waitretry=3 https://github.com/sgerrand/alpine-pkg-glibc/releases/download/${GLIBC_VERSION}/${pkg}.apk -O /tmp/${pkg}.apk && \
                    apk add --no-cache /tmp/${pkg}.apk && rm -f /tmp/${pkg}.apk; \
                done && rm -f /etc/apk/keys/sgerrand.rsa.pub && \
                /usr/glibc-compat/bin/localedef --force --inputfile POSIX --charmap UTF-8 ${LANG} || true && \
                echo "export LANG=${LANG}" > /etc/profile.d/locale.sh && \
                /usr/glibc-compat/sbin/ldconfig /lib /usr/glibc-compat/lib && \
                echo 'hosts: files mdns4_minimal [NOTFOUND=return] dns mdns4' >> /etc/nsswitch.conf && \
                rm -f /root/.wget-hsts && apk-remove .build-dependencies wget ca-certificates glibc-i18n && \
                apk-cleanup


