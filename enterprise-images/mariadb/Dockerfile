#
# MAINTAINER				Carson,C.J.Zeong <zcy@nicescale.com>
# DOCKER_VERSION			1.11.1
#
# Dockerizing Mariadb: Dockerfile for building Mariadb images
#
FROM zcy/centos7
MAINTAINER Carson,C.J.Zeong <zcy@nicescale.com>

ENV DATA_DIR /var/lib/mysql
ENV TERM xterm
 
RUN yum -y install mariadb mariadb-server && \
    yum clean all

COPY mysqld_charset.cnf /etc/my.cnf.d/
COPY my.cnf /etc/my.cnf
COPY scripts /scripts

RUN chmod +x /scripts/start

# EXPOSE 3306

CMD ["/scripts/start"]