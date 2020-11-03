FROM tomcat:9.0-jdk11-adoptopenjdk-hotspot
# cannot make it works yet #LABEL groupId=${project.groupId} artifactId=${project.artifactId} version=${project.version}

RUN cp -r  /usr/local/tomcat/webapps.dist/examples/ /usr/local/tomcat/webapps/examples
RUN mkdir /usr/local/tomcat/webapps/khoolsession &&\
    chmod g+rwx /usr/local/tomcat/webapps/khoolsession
ADD conf/conf-tomcat-server.xml /usr/local/tomcat/conf/server.xml
ADD conf/conf-context.xml /usr/local/tomcat/conf/context.xml

ADD khoolsession-0.3-SNAPSHOT.war /usr/local/tomcat/webapps/khoolsession.war
