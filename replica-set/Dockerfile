# mongo-cluster
# replica-set part
FROM mongo:latest

EXPOSE 27017

COPY source.list /etc/apt/source.list

RUN mkdir -p /opt/mongo/ && \
	apt-get -y update && apt-get -y upgrade && apt-get install -y netcat

COPY start_replica.sh /opt/mongo/
COPY init.sh /opt/mongo/

WORKDIR /opt/mongo

CMD /opt/mongo/start_replica.sh

