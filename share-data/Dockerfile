FROM testthedocs/ttd-caddy

LABEL maintainer "wundertax devops <devops@wundertax.de>" \
    org.label-schema.vendor = "wundertax"

COPY Caddyfile /etc/Caddyfile
COPY html /srv/html/

#RUN mkdir -p /srv/logs \
#    && touch /srv/logs/access.log

EXPOSE 80 443 2015
#WORKDIR /srv

ENTRYPOINT ["/sbin/tini", "--", "/usr/bin/caddy"]
CMD ["--conf", "/etc/Caddyfile"]

