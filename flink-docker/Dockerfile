################################################################################
#  Licensed to the Apache Software Foundation (ASF) under one
#  or more contributor license agreements.  See the NOTICE file
#  distributed with this work for additional information
#  regarding copyright ownership.  The ASF licenses this file
#  to you under the Apache License, Version 2.0 (the
#  "License"); you may not use this file except in compliance
#  with the License.  You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
# limitations under the License.
################################################################################

FROM java:8-jre-alpine

# Install requirements
RUN apk add --no-cache bash snappy

# Flink environment variables
ENV FLINK_INSTALL_PATH=/opt
ENV FLINK_HOME $FLINK_INSTALL_PATH/flink
ENV PATH $PATH:$FLINK_HOME/bin

# flink-dist can point to a directory or a tarball on the local system
ARG flink_dist=NOT_SET

# Install build dependencies and flink
ADD $flink_dist $FLINK_INSTALL_PATH

##should we add volumn to represent blob.storage.directory: /flink-blob/
RUN set -x && mkdir /flink-blob && chmod -R 777 /flink-blob && \
  ln -s $FLINK_INSTALL_PATH/flink-* $FLINK_HOME && \
  addgroup -S hdfs && adduser -D -S -H -G hdfs -h $FLINK_HOME hdfs && \
  chown -R hdfs:hdfs $FLINK_INSTALL_PATH/flink-* && \
  chown -h hdfs:hdfs $FLINK_HOME


COPY docker-entrypoint.sh /
RUN chmod +x docker-entrypoint.sh

USER hdfs
EXPOSE 8081 6123
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["--help"]