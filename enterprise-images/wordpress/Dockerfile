FROM zcy/php-fpm

ADD init.sh /init.sh

CMD ["/init.sh", "/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
