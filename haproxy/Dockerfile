FROM golang:1.14 AS go_builder
WORKDIR /go/src/github.com/percona/percona-xtradb-cluster-operator/src

RUN go get k8s.io/apimachinery/pkg/util/sets \
    && curl -Lf -o /go/src/github.com/percona/percona-xtradb-cluster-operator/src/peer-list.go https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/master/cmd/peer-list/main.go \
    && go build peer-list.go

FROM centos:8

LABEL org.opencontainers.image.authors="info@percona.com"

RUN groupadd -g 1001 mysql
RUN useradd -u 1001 -r -g 1001 -s /sbin/nologin \
        -c "Default Application User" mysql


# check repository package signature in secure way
RUN export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver pool.sks-keyservers.net --recv-keys 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A 99DB70FAE1D7CE227FB6488205B555B38483C65D \
	&& gpg --export --armor 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A > ${GNUPGHOME}/RPM-GPG-KEY-Percona \
	&& gpg --export --armor 99DB70FAE1D7CE227FB6488205B555B38483C65D > ${GNUPGHOME}/RPM-GPG-KEY-Official-Key \
	&& rpmkeys --import ${GNUPGHOME}/RPM-GPG-KEY-Percona ${GNUPGHOME}/RPM-GPG-KEY-Official-Key \
	&& curl -Lf -o /tmp/percona-release.rpm https://repo.percona.com/yum/percona-release-latest.noarch.rpm \
	&& rpmkeys --checksig /tmp/percona-release.rpm \
	&& rpm -i /tmp/percona-release.rpm \
	&& rm -rf "$GNUPGHOME" /tmp/percona-release.rpm \
	&& rpm --import /etc/pki/rpm-gpg/PERCONA-PACKAGING-KEY \
	&& dnf module -y disable mysql \
	&& percona-release disable all \
	&& percona-release setup ps80

# install exact version of PS for repeatability
ENV PERCONA_VERSION 8.0.19-10.1.el8

RUN set -ex; \
    dnf install -y \
        haproxy \
        percona-server-client-${PERCONA_VERSION} \
        which \
        tar \
        policycoreutils; \
    \
    dnf clean all; \
    rm -rf /var/cache/dnf

STOPSIGNAL SIGUSR1

RUN set -ex; \
    mkdir -p /etc/haproxy/pxc /etc/haproxy-auto; \
    chown -R 1001:1001 /run /etc/haproxy /etc/haproxy/pxc /etc/haproxy-auto

COPY dockerdir/bin/add_pxc_nodes.sh /usr/bin/add_pxc_nodes.sh
COPY dockerdir/bin/entrypoint.sh /
COPY dockerdir/etc/haproxy.cfg /etc/haproxy/
COPY dockerdir/etc/haproxy-auto.cfg /etc/haproxy
COPY dockerdir/bin/check_pxc.sh /usr/local/bin
COPY --from=go_builder /go/src/github.com/percona/percona-xtradb-cluster-operator/src/peer-list /usr/bin/

RUN set -ex; \
    chown 1001:1001 /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy-auto.cfg /usr/local/bin/check_pxc.sh

USER mysql

VOLUME ["/etc/haproxy/pxc"]

ENTRYPOINT ["/entrypoint.sh"]
CMD ["haproxy"]
