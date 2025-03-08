#this is multi stage docker file
#first stage is to build the project and second stage is to deploy the war file to tomcat
#this is a simple example of multi stage docker file
#this build the artifact and then deploy it to tomcat
FROM  maven:3.9.9-eclipse-temurin-21-jammy AS build_image
RUN git clone https://github.com/hkhcoder/vprofile-project.git
RUN cd vprofile-project && git checkout containers && mvn clean install -DskipTests
#this will build the project and create the war file in the target folder
#second stage
FROM tomcat:10-jdk21
RUN rm -rf /usr/local/tomcat/webapps/*
COPY --from=build_image   vprofile-project/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
#/vprofile-project/vprofile-web/target/vprofile-web.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
