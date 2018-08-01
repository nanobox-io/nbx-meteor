FROM ubuntu

# Nanobox configuration
# 
# route http traffic to port 80 within the container
LABEL io.nanobox.http_port="80"
# stream service logs
LABEL io.nanobox.logs.1.path="/var/log/runit/app/current"
LABEL io.nanobox.logs.1.prefix="app[out]"

# specify the node version
ARG NODE_VERSION=v8.11.1

# update ubuntu package database
RUN apt-get update -q && apt-get clean

# install dependencies
RUN apt-get install -y \
      runit \
      curl

# install meteor - meteor installer doesn't work with the default tar binary
# see https://github.com/coreos/bugs/issues/1095
RUN apt-get install -y bsdtar \
    && cp $(which tar) $(which tar)~ \
    && ln -sf $(which bsdtar) $(which tar)

# install Meteor forcing its progress
RUN curl "https://install.meteor.com/" | sh

# put back the original tar
RUN mv $(which tar)~ $(which tar)

# install node
RUN \
    # fetch the nodejs version as specified in the ARG above ^
    curl https://nodejs.org/dist/$NODE_VERSION/node-$NODE_VERSION-linux-x64.tar.gz \
      > /tmp/required-node-linux-x64.tar.gz && \
      
    # extract the node tarball
    cd /usr/local && \
    tar \
      --strip-components 1 \
      -xzf /tmp/required-node-linux-x64.tar.gz && \
      
    # cleanup
    rm /tmp/required-node-linux-x64.tar.gz
    

# setup runit service definition:
RUN mkdir -p /etc/service/app
RUN mkdir -p /etc/service/app/log
COPY .nbx/web/runit/app /etc/service/app/run
COPY .nbx/web/runit/app-log /etc/service/app/log/run
  
# bring in the source code
ADD . ./src

# build the meteor app
RUN cd /src && \
    meteor build /app \
      --directory \
      --allow-superuser

# verify the correct version of node is installed
RUN bash -c 'if [ "$NODE_VERSION" != "$(</app/bundle/.node_version.txt)" ]; then \
      echo "================== ERROR =================="; \
      echo ""; \
      echo "NODE_VERSION does not match the node version required from the build."; \
      echo "Please update the NODE_VERSION in the dockerfile to match: $(</app/bundle/.node_version.txt)"; \
      echo ""; \
      exit 1; \
    fi'
    
# install packages
RUN cd /app/bundle/programs/server && \
    npm install
    
# cleanup
RUN rm /usr/local/bin/meteor && \
    rm -rf ~/.meteor && \
    rm -rf /src && \
    apt-get --purge autoremove curl -y

# run runit services
CMD ["runsvdir", "-P", "/etc/service"]
