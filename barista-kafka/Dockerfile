FROM redhat-openjdk18-openshift:lastest
LABEL org.label-schema.description="" org.label-schema.version=1.0-SNAPSHOT org.label-schema.schema-version=1.0 org.label-schema.build-date=2020-02-28T15:10:26.519 org.label-schema.name=barista-kafka org.label-schema.vcs-ref=5f50e7ea547cf54b4c3f4f8e2d03f60692053bd0 org.label-schema.vcs-url=https://github.com/zhaoxiyi/quarkus-coffeeshop-demo.git
EXPOSE 8080
COPY maven /opt/
WORKDIR /opt
ENTRYPOINT ["java","-Dquarkus.http.host=0.0.0.0","-jar","barista-kafka-1.0-SNAPSHOT-runner.jar"]
