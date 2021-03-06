FROM centos:7
# MAINTAINER Percona Development <info@percona.com>
MAINTAINER Reza Farrahi <reza.farrahi@nakisa.com>

#
RUN groupadd -g 5341 -r mysql
RUN useradd -m -s /sbin/nologin -u 999 -g mysql -c "Default Application User" mysql
COPY permission_fix /usr/local/bin/
RUN chmod 755 /usr/local/bin/permission_fix
RUN permission_fix

# check repository package signature in secure way
RUN export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver ha.pool.sks-keyservers.net --recv-keys 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A \
	&& gpg --export --armor 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A > ${GNUPGHOME}/RPM-GPG-KEY-Percona \
	&& rpmkeys --import ${GNUPGHOME}/RPM-GPG-KEY-Percona /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7 \
	&& curl -L -o /tmp/percona-release.rpm https://repo.percona.com/centos/7/RPMS/noarch/percona-release-0.1-8.noarch.rpm \
	&& rpmkeys --checksig /tmp/percona-release.rpm \
	&& yum install -y /tmp/percona-release.rpm \
	&& rm -rf "$GNUPGHOME" /tmp/percona-release.rpm \
	&& rpm --import /etc/pki/rpm-gpg/PERCONA-PACKAGING-KEY \
	&& percona-release disable all \
	&& percona-release enable ps-80 release \
	&& percona-release enable tools release

#	&& rpmkeys --checksig /tmp/percona-release.rpm \
#	&& curl -L -o /tmp/percona-release.rpm https://repo.percona.com/centos/7/RPMS/noarch/percona-release-1.0-7.noarch.rpm \
#	&& curl -L -o /tmp/percona-release.rpm https://repo.percona.com/centos/7/RPMS/noarch/percona-release-0.1-8.noarch.rpm \

# install exact version of PS for repeatability
ENV PERCONA_VERSION 8.0.13-4.1.el7

RUN yum install -y \
		percona-server-server-${PERCONA_VERSION} \
		percona-server-tokudb-${PERCONA_VERSION} \
		percona-server-rocksdb-${PERCONA_VERSION} \
		percona-xtrabackup-80 \
		which \
		wget \
		nc \
		policycoreutils \
		https://repo.percona.com/percona/yum/release/7/RPMS/x86_64/jemalloc-3.6.0-1.el7.x86_64.rpm \
	&& yum clean all \
	&& rm -rf /var/cache/yum /var/lib/mysql

# purge and re-create /var/lib/mysql with appropriate ownership
RUN /usr/bin/install -m 0775 -o mysql -g mysql -d /var/lib/mysql /var/run/mysqld /docker-entrypoint-initdb.d \
# comment out a few problematic configuration values
	&& find /etc/my.cnf /etc/my.cnf.d -name '*.cnf' -print0 \
		| xargs -0 grep -lZE '^(bind-address|log|user)' \
		| xargs -rt -0 sed -Ei 's/^(bind-address|log|user)/#&/' \
# don't reverse lookup hostnames, they are usually another container
	&& echo '!includedir /etc/my.cnf.d' >> /etc/my.cnf \
	&& printf '[mysqld]\nskip-host-cache\nskip-name-resolve\n' > /etc/my.cnf.d/docker.cnf \
# TokuDB modifications
	&& /usr/bin/install -m 0664 -o mysql -g mysql /dev/null /etc/sysconfig/mysql \
	&& echo "LD_PRELOAD=/usr/lib64/libjemalloc.so.1" >> /etc/sysconfig/mysql \
	&& echo "THP_SETTING=never" >> /etc/sysconfig/mysql \
# allow to change config files
	&& chown -R mysql:mysql /etc/my.cnf /etc/my.cnf.d \
	&& chmod -R ug+rwX /etc/my.cnf /etc/my.cnf.d

#
VOLUME ["/var/lib/mysql", "/var/log/mysql"]

#
RUN chown -R mysql:mysql /var/lib/mysql \
	&& chmod -R ug+rwX /var/lib/mysql

#
COPY ps-entry.sh /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]

#
USER mysql
# USER root
EXPOSE 3306 33060
CMD ["mysqld"]
