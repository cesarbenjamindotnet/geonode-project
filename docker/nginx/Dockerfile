ARG BASE_IMAGE_VERSION
FROM geonode/nginx:${BASE_IMAGE_VERSION}

WORKDIR /etc/nginx/

ADD geonode.conf.envsubst ./sites-enabled/
RUN rm -rf /mnt/volumes/statics/atlas/
ADD atlas /mnt/volumes/statics/
