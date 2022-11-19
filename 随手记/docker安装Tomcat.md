docker pull tomcat:7.0.106

docker run --name tomcat  --restart=always -p 8080:8080 -v D:/tomcat/webapps:/usr/local/tomcat/webapps -v D:/tomcat/conf:/usr/local/tomcat/conf -d tomcat:7.0.106



docker run --name tomcat-text
docker cp tomcat-text:/usr/local/tomcat/conf D:/tomcat/conf
docker cp tomcat-text:/usr/local/tomcat/webapps.dist D:/tomcat/webapps
