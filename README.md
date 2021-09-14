# HyperFrameOE-WildFly
- 업로드된 바이너리는 HyperFrame Open Edition WildFly 제품 설치를 위한 파일  
- 웹서버와 연동하는데 필요한 mod_jk.so 파일은 HyperFrameOE-Apache의 binaries 폴더에서도 확인 가능

## 설치 파일

### WildFly
* Verison : wildfly-22.0.0
* Note : https://www.wildfly.org/downloads/

## 검증 환경

* CentOS Linux release 7.9.2009
* CentOS Linux release 8.4.2105
* Ubuntu 20.04.1 LTS

## 설치 및 실행

### 1) WildFly 압축 풀기

    $ tar -zxf wildfly-22.0.0.tar.gz

### 2) 디렉토리 구조 확인

      Wildfly
      ├── standalone        
      ├── applicient      
      ├── bin             
      ├── docs             
      ├── welcome-content
      ├── moduels          
      ├── domain          
      └── tmp               
      
### 3) Port 확인 및 변경

    $ vi ${WILDFLY_HOME}/standalone/standalone.xml

    # 프로토콜에 따른 Port 확인 및 변경
    <socket-binding-group name="standard-sockets" default-interface="public" port-offset="${jboss.socket.binding.port-offset:0}">
        <socket-binding name="ajp" port="${jboss.ajp.port:8009}"/>
        <socket-binding name="http" port="${jboss.http.port:8080}"/>
        <socket-binding name="https" port="${jboss.https.port:8443}"/>
        ...
    </socket-binding-group>
      
### 4) WildFly 실행

    $ vi ${WILDFLY_HOME}/bin/
    $ ./standalone.sh

### 5) WildFly 종료 

* 관리자 접속 / 종료

      $ cd ${WILDFLY_HOME}/bin/
      $ ./jboss-cli.sh 
      You are disconnected at the moment. Type 'connect' to connect to the server or 'help' for the list of supported commands.
      [disconnect /] connect localhost:[manage port]                    # standalone.xml 확인 가능
      [standalone@192.168.3.87:9990 /] shutdown                         # Admin Mode

* 한 줄 종료

      $ cd ${WILDFLY_HOME}/bin/
      $ ./jboss-cli.sh --controller:localhost:[manage port] --connect command=:shutdown

## 버전 확인

    $ cd ${WILDFLY_HOME}/bin/
    $ ./standalone.sh --version

    =========================================================================

      JBoss Bootstrap Environment

      JBOSS_HOME: ${WILDFLY_HOME}

      JAVA: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.302.b08-0.el7_9.x86_64/jre/bin/java

      JAVA_OPTS:  -server -Xms64m -Xmx512m -XX:MetaspaceSize=96M -XX:MaxMetaspaceSize=256m -Djava.net.preferIPv4Stack=true -Djboss.modules.system.pkgs=org.jboss.byteman -Djava.awt.headless=true

    =========================================================================

    00:00:00,479 INFO  [org.jboss.modules] (main) JBoss Modules version 1.11.0.Final
    WildFly Preview 22.0.0.Final (WildFly Core 14.0.0.Final)
    
## 로그 정보

### 1) 로그 경로

    ${WILDFLY_HOME}/standalone/log/server.log

### 2) 로그 경로 변경
    
    # Log Path 추가
    $ vi ${WILDFLY_HOME}/bin/standalone.sh
    ...
    export JAVA_OPTS=" $JAVA_OPTS -Djboss.server.log.dir=[Log Path]"
    ...

## 환경 설정 파일 정보

### 1) 환경 설정 파일 경로

    ${WILDFLY_HOME}/standalone/configuration/standalone.xml

### 2) 환경 설정

* 외부 Client 접속 허용
      
      ...
      <interface name="public">
          <!--<inet-address value="${jboss.bind.address:192.168.3.87}"/>-->   # 주석 처리 및 삭제
          <any-address/>                                                      # 추가
      </interface>
      ...
      
* port 변경

      ... 
      <socket-binding-group name="standard-sockets" default-interface="public" port-offset="${jboss.socket.binding.port-offset:0}">
          <socket-binding name="ajp" port="${jboss.ajp.port:8009}"/>
          <socket-binding name="http" port="${jboss.http.port:8080}"/>
          <socket-binding name="https" port="${jboss.https.port:8443}"/>
          <socket-binding name="management-http" interface="management" port="${jboss.management.http.port:9990}"/>
          <socket-binding name="management-https" interface="management" port="${jboss.management.https.port:9993}"/>
      ...

## Web Server 연동

### 1) Apache

* 사전에 apache 설치가 필요 (Apache 제품의 README.MD 파일 참고)

* tomcat-connectors-1.2.46-src.tar.gz 압축 해제, tomcat connector jk 설치

      $ tar xzvf tomcat-connectors.1.2.46-src.tar.gz
      $ cd tomcat-connectors.1.2.46-src/native
      $ ./configure --with-apxs=/${APACHE_HOME}/bin/apxs
      $ make & make install

* 정상 설치 완료 시, ${APACHE_HOME}/modules/에서 mod_jk.so 파일 생성

* ${APACHE_HOME}/conf/httpd.conf 파일에 mod_jk 설정 추가

      ...
      LoadModule jk_module modules/mod_jk.so
      Include conf/extra/mod_jk.conf
      ...

* ${APACHE_HOME}/conf/extra/mod_jk.conf 파일 생성

      $ vi ${APACHE_HOME}/conf/extra/mkd_jk.conf
      
      # mkd_jk.conf
      <IfModule mod_jk.c>
      JkWorkersFile conf/workers.properties
      JkShmFile logs/mod_jk.shm
      JkLogFile logs/mod_jk.log
      JkLogLevel info
      JkLogStampFormat "[%a %b %d %H:%M:%S %Y] "
      JkMountFile conf/uriworkermap.properties
      </IfModule>
      
* ${APACHE_HOME}/conf/workers.properties 파일 생성, 연동 Tomcat List 작성

      $ vi ${APACHE_HOME}/conf/workers.properties
      
      # workers.properties
      worker.list=worker1
      worker.worker1.port=8009
      worker.worker1.host=192.168.0.120
      worker.worker1.type=ajp13

* ${APACHE_HOME}/conf/uriworkermap.properties 파일 생성, Tomcat에서 처리할 uri 작성

      /*=worker1

* ${WILDFLY_HOME}/standalone/configure/standalone.xml Web Server 통신 Listener 추가

      ...
      <server name="default-server">
          ...
          <ajp-listener name="ajp" socket-binding="ajp" />
          ...
      </server>
      ...
      
* Apache 및 WildFly 기동
      
### 2) Nginx

* 사전에 Nginx 설치가 필요 (Nginx 제품의 README.MD 파일 참고)

* [Nginx] ${NGINX_HOME}/conf/nginx.conf 파일 수정
      
      $ vi ${NGINX_HOME}/conf/nginx.conf
      ...
      location / {
           root             html;
           index            index.html index.htm;
           proxy_pass     http://[WildFly IP]:[WildFly Listen Port];
      }
      ...

* [WildFly] ${WILDFLY_HOME}/standalone/configuration/standalone.xml 파일 수정
      
      
      $ vi ${WILDFLY_HOME}/standalone/configuration/standalone.xml
      ...
      <inferface>
           <interface name="management">
                <inet-address value="${jboss.bind.address.management:127.0.0.1}"/>
           </interface>
                <interface name="public">
                <any-address/>
        </interface>
      ...

      ...
      <server name="default-server">
           <http-listener name="default" socket-binding="http" redirect-socket="https" enable-http2="true" proxy-address-forwarding="true"/>
      ...
      </server>
 
* Nginx 및 WildFly 기동 
