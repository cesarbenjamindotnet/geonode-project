ARG BASE_IMAGE_VERSION
FROM geonode/nginx:${BASE_IMAGE_VERSION}

WORKDIR /etc/nginx/

ADD geonode.conf.envsubst ./sites-enabled/

WORKDIR /
RUN rm -rf /mnt/volumes/statics/atlas/
RUN mkdir -p /mnt/volumes/statics/atlas/
COPY atlas /mnt/volumes/statics/atlas/
