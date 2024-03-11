ARG BASE_IMAGE_VERSION
FROM geonode/geoserver_data:${BASE_IMAGE_VERSION}

ADD ./anvcc_styles/* /geoserver_data/data/styles/

RUN mkdir -p /geoserver_data/data/data/
COPY ./anvcc_data/* /geoserver_data/data/data/