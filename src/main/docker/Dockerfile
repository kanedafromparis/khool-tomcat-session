FROM tomcat:9.0.36-jdk11-adoptopenjdk-hotspot
LABEL groupId=${project.groupId} artifactId=${project.artifactId} version=${project.version}

RUN cp -r  /usr/local/tomcat/webapps.dist/examples/ /usr/local/tomcat/webapps/examples
RUN mkdir /usr/local/tomcat/webapps/${project.artifactId} && chmod g+rwx /usr/local/tomcat/webapps/${project.artifactId}
ADD maven/conf/conf-tomcat-server.xml /usr/local/tomcat/conf/server.xml
ADD maven/conf/conf-context.xml /usr/local/tomcat/conf/context.xml

ADD maven/war/${project.artifactId}-${project.version}.war /usr/local/tomcat/webapps/khoolsession.war
