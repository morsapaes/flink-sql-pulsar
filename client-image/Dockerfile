###############################################################################
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
###############################################################################

FROM flink:1.12.0-scala_2.11

# Copy sql-client script
COPY sql-client/ /opt/sql-client
RUN mkdir -p /opt/sql-client/lib

# Download connector libraries
RUN wget -P /opt/sql-client/lib/ https://repo.maven.apache.org/maven2/org/apache/flink/flink-json/1.12.0/flink-json-1.12.0.jar; \
	wget -P /opt/sql-client/lib/ https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-avro/1.12.0/flink-sql-avro-1.12.0.jar; \
	wget -P /opt/sql-client/lib/ https://repo.maven.apache.org/maven2/org/apache/avro/avro/1.10.0/avro-1.10.0.jar; \
    wget -P /opt/sql-client/lib/ https://dl.bintray.com/streamnative/maven/io/streamnative/connectors/pulsar-flink-connector-2.11-1.12/2.7.0/pulsar-flink-connector-2.11-1.12-2.7.0.jar;

# Copy configuration
COPY conf/* /opt/flink/conf/

WORKDIR /opt/sql-client
ENV SQL_CLIENT_HOME /opt/sql-client

COPY docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
