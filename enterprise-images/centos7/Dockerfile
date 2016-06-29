#
# MAINTAINER				Carson,C.J.Zeong <zcy@nicescale.com>
# DOCKER_VERSION			1.11.1
#
# Dockerizing Centos7: Dockerfile for building CentOS images
#
FROM centos:7.2.1511
MAINTAINER Carson,C.J.Zeong <zcy@nicescale.com>

ENV TZ "Asia/Shanghai"

RUN yum -y install curl wget tar bzip2 unzip vim-enhanced passwd sudo yum-utils hostname net-tools rsync man && \
    yum install -y gcc gcc-c++ git make automake cmake patch python-devel libpng-devel libjpeg-devel epel-release && \
    yum install -y --enablerepo=epel pwgen python-pip && \
    yum clean all

RUN pip install supervisor && \
    mkdir -p /etc/supervisor/confd && \
    mkdir -p /var/log/supervisor

COPY supervisord.conf /etc/supervisord.conf

# EXPOSE 22

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]