  ### Introduction
If we want to run cron job inside the container to call the java jar file then we need to setup the project as mentioned below, but first lets understand the project structure.
```java
project_name
  -src
    -main
      -docker
        *crontab.txt
        *docker-assembly.xml
        *Dockerfile
      -java
  -target
    -classes
    -docker
      ....
        - build
          -maven
            - project_name.jar
    - project_name.jar
  -.gitignore
  -pom.xml
```
Above skeleton is maven java project where we added a docker folder inside main to add all docker related files. Dockerfile can only read files from this base directory so it is important to place all files which needs to be copied over to docker container in the same base directory.

### Typical Cron

1. List all cron
```java
crontab -l
```
2. Configuring new cron job
```java
#edit crontab
crontab -e

#add below line
* * * * * /PATH_TO_SCRIPT/script1.sh >> /PATH_TO_SCRIPT_LOG/scheduler_log.txt 2>&1
/* minute  hour  day_of_month  month  weekday  command */
```
3. Script called from cron
```java
#!/bin/bash
# calling jar file
java -jar /PATH_TO_JAR/project_name.jar >> /PATH_TO_SCRIPT_LOG/scheduler_log.txt
```

### Part by Part for Docker
1. crontab.txt
```java
*  *  * * *  root /usr/lib/jvm/jdk1.8.0_111/bin/java -jar /opt/project_dump/project_name.jar >>/var/log/cron.log 2>&1
#empty
```
In defining crontab if we don't add #empty line after the command it fails to execute in the container. Also in this case we are calling the jar file directly from the crontab instead of calling via seperating script_file.sh etc. If in case `java command not found` is thrown upon executing `* * * * * root java -jar project_name.jar` we need to add the complete java jre or jdk path as shown above.

use below if we want to test by echoing message
```java
*  *  * * *  root echo "Hello World" >>/var/log/cron.log 2>&1
#empty
```

2. Dockerfile
```java
FROM java:1.8

#Create directory for all files
RUN mkdir /opt/project_dump/

#Add crontab.txt to folder
ADD crontab.txt /opt/project_dump/crontab.txt

#ADD built .jar to folder
ADD maven/project_name.jar /opt/project_dump/project_name.jar

#install and create logfile
RUN yum -y install cronie
RUN touch /var/log/cron.log

#set crontab from crontab.txt file
ADD crontab.txt /etc/cron.d/crontab

#change permissions
RUN chmod 0644 /etc/cron.d/crontab

# start crond in foreground
CMD ["crond","-n"]
```
In maven project .jar file is created inside the target folder however if we have Dockerfile is present inside src/main/docker/ then it wouldn't find other parent directory than docker folder. So everything needs to be defined inside the docker folder. The solution to this is grabbing .jar file inside `maven` folder created the docker-maven-plugin.

3. docker-assembly.xml
```java
<assembly>
    <fileSets>
        <fileSet>
            <directory>${project.build.directory}</directory>
            <filtered>true</filtered>
            <fileMode>0755</fileMode>
            <includes>
                <include>project_name.jar</include>
            </includes>
            <outputDirectory>.</outputDirectory>
        </fileSet>
    </fileSets>
</assembly>
```
4. pom.xml
The main part about docker-maven-plugin is referring docker-assembly and Dockerfile from the pom. Below is docker related pom details
```xml
<properties>
   <docker-maven.version>0.20.0</docker-maven.version>
</properties>
<build>
 <plugins>
   <plugin>
      <groupId>io.fabric8</groupId>
      <artifactId>docker-maven-plugin</artifactId>
      <version>${docker-maven.version}</version>
      <configuration>
        <images>
          <image>
            <name>
              project_name
            </name>
                <build>
                  <!-- fail the build if old images can't be removed -->
                  <cleanup>remove</cleanup>
                  <assembly>
                    <basedir>/opt/project_dump</basedir>
                    <descriptor>docker-assembly.xml</descriptor>
                  </assembly>
                  <!-- referencing to use our own Dockerfile -->
                  <dockerFile>${project.basedir}/src/main/docker/Dockerfile</dockerFile>
                </build>
                <run>
                  <!-- if we need to refer environment variable in .jar file -->
                  <envPropertyFile>${project.basedir}/local/local.properties</envPropertyFile>
                </run>
            </image>
        </images>
      </configuration>
    </plugin>
 </plugins>
</build>
```
### Docker Build and RUN

1.Build and Run
```shell
mvn package docker:build docker:run  
```
docker:build and docker:run is valid if docker-maven-plugin is installed. Once the build is successful we should get CONTAINER_ID.

2. Check for Running container
```shell
docker ps
```
If the entrypoint in plugin (pom.xml) or CMD doesn't run in foreground then the container will run and exit immediately after executing the command. So we are using `CMD ["crond", "-n"]` in the Dockerfile.

3. Check inside container
```shell
docker exec -it CONTAINER_ID bash
```
this should give us access to the running container and we can verify if the .jar, files are copied to opt/project_dump folder and cron.log is updated with the logs.

4. Spinning up new container from built image
```shell
docker images #get latest image id
docker run -it IMAGE_ID bash
```
