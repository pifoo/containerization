#
# MAINTAINER				Carson,C.J.Zeong <zcy@nicescale.com>
# DOCKER_VERSION			1.11.1
#
# Dockerizing Nginx: Dockerfile for building Nginx images
#

FROM zcy/centos7
MAINTAINER Carson,C.J.Zeong <zcy@nicescale.com>

ENV APP_DIR /app

RUN yum -y install nginx php-cli php-mysql php-pear php-ldap php-mbstring php-soap php-dom php-gd php-xmlrpc php-fpm php-mcrypt && \ 
	yum clean all

RUN sed -i 's/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/' /etc/php.ini && \
    mkdir -p /app && echo "<?php phpinfo(); ?>" > ${APP_DIR}/info.php

COPY nginx_nginx.conf /etc/nginx/nginx.conf
COPY nginx_default.conf /etc/nginx/conf.d/default.conf
COPY php_www.conf /etc/php-fpm.d/www.conf
COPY app/ /app

COPY supervisor_nginx.conf /etc/supervisor/confd/nginx.conf
COPY supervisor_php-fpm.conf /etc/supervisor/confd/php-fpm.conf

# EXPOSE 80 443

ONBUILD COPY . /app
ONBUILD RUN chown -R nginx:nginx  /app