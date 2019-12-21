FROM harbor.devops.com/devops/tomcat:8
RUN rm -rf /usr/local/tomcat/webapps/* && mkdir /usr/local/tomcat/webapps/ROOT -p
COPY ./ROOT /usr/local/tomcat/webapps/ROOT
CMD [ "sh", "-c", "catalina.sh run" ]
