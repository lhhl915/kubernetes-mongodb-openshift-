# mongo-cluster
# configsvr part
FROM mongo:latest

EXPOSE 27017

RUN mkdir -p /opt/mongo/ && \
	apt-get update && apt-get install -y netcat

COPY start_replica.sh /opt/mongo/

RUN chown -R root.root /data

VOLUME /data

WORKDIR /opt/mongo

CMD /opt/mongo/start_replica.sh
