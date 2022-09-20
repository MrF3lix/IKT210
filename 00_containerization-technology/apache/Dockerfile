FROM httpd:2.4.54-alpine

COPY apache.conf /usr/local/apache2/conf/extra/httpd-vhosts.conf
COPY httpd.conf /usr/local/apache2/conf/httpd.conf

CMD ["httpd-foreground"]
