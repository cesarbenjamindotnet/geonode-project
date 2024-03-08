ARG BASE_IMAGE_VERSION
FROM geonode/nginx:${BASE_IMAGE_VERSION}

WORKDIR /etc/nginx/

ADD geonode.conf.envsubst ./sites-enabled/
RUN rm -r dir /mnt/volumes/statics/atlas/
ADD atlas /mnt/volumes/statics/
