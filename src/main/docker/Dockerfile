FROM tomcat:jdk11-openjdk-slim
LABEL groupId=${project.artifactId} artifactId=${project.artifactId} version=${project.version}

RUN cp -r  /usr/local/tomcat/webapps.dist/examples/ /usr/local/tomcat/webapps/examples
ADD maven/conf/conf-tomcat-server.xml /usr/local/tomcat/conf/server.xml
ADD maven/conf/conf-context.xml /usr/local/tomcat/conf/context.xml

ADD maven/war/${project.artifactId}-${project.version}.war /usr/local/tomcat/webapps/khoolsession.war
