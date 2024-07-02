ARG BASE_IMAGE_VERSION
FROM geonode/geoserver_data:${BASE_IMAGE_VERSION}

COPY styles/* /geoserver_data/data/styles/

# RUN mkdir -p /geoserver_data/data/data/
COPY data/* /geoserver_data/data/data/

COPY workspaces/anvcc/ /geoserver_data/data/workspaces/anvcc/
# COPY styles/* /geoserver_data/data/styles/
