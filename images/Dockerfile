
FROM boostport/hbase-phoenix-all-in-one
LABEL description="Mini Python Ingester from Kafka to Phoenix/HBase"

ADD ./* /opt/
RUN mv /opt/hbase-site.xml /opt/phoenix-server/bin
RUN sh /opt/zoo_config.sh


#CMD ["./zoo_config.sh"]