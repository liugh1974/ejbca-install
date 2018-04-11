# EJBCA INSTALL

ejbca_ce_6_10_1_2 + wildfly-10.1.0.Final + JDK8(Oracle) + mysql安装步骤：

(以下步骤我亲自安装测试了数十次，确定每次都可以成功)


# 1. 下载ejbca_ce_6_10_1_2, wildfly-10.1.0.Final, JDK8 和 ANT
   
    ejbca_ce_6_10_1_2： https://sourceforge.net/projects/ejbca/files/ejbca6/ejbca_6_10_0/ejbca_ce_6_10_1_2.zip/download
    
    wildfly-10.1.0.Final： http://download.jboss.org/wildfly/10.1.0.Final/wildfly-10.1.0.Final.zip
    
    JDK8: http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
    
    ANT: http://mirrors.shu.edu.cn/apache//ant/binaries/apache-ant-1.10.3-bin.zip
 
# 2. wildfly解压后，需要在 /home/your-username/.bashrc 或 /etc/profile 添加以下配置
	
	以 /home/xxxooo/jboss/wildfly 目录为例
	
	export JBOSS_HOME=/home/xxxooo/jboss/wildfly
	export APPSRV_HOME=/home/xxxooo/jboss/wildfly
	export PATH=$PATH:$JBOSS_HOME/bin
	
# 3. 安装好 JDK 后，需要安装更新 JDK 的 Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files
   
   	如果不更新的话，由于美国法律对相关加密算法出口限制，密码长度会有限制，最多不能超过7位

   	下载地址：http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html
	 
	 下载完成解压会得到 local_policy.jar， US_export_policy.jar 两个jar文件
	 把这两个jar文件替换 JDK 的相同的文件。
	 替换文件目录： $JAVA_HOME/jre/lib/security
	

# 4. 配置wildfly数据源
  
	(数据源配置本来是可以用jboss-cli命令来做，但是用jboss-cli命令来做的话，在后面第12步的时候有时候会把添加的数据源删除掉，不知道什么原因，所以还是手动添加)

## 1). 在 $WILDFLY_HOME modules/system/layers/base/com 目录新建 mysql/jdbc/main目录;
## 2). 复制mysql驱动jar文件到modules/system/layers/base/com/mysql/jdbc/main目录。在此我们以mysql-connector-java-5.1.42.jar为例;
## 3). 在 modules/system/layers/base/com/mysql/jdbc/main目录新建一个文件 module.xml
		
	module.xml 文件内容:
		
	<?xml version="1.0" encoding="UTF-8"?>
	<module xmlns="urn:jboss:module:1.3" name="com.mysql.jdbc">
		<resources>
			<resource-root path="mysql-connector-java-5.1.42.jar"/>
		</resources>
		<dependencies>
			<module name="javax.api"/>
			<module name="javax.transaction.api"/>
			<module name="javax.servlet.api" optional="true"/>
		</dependencies>
	</module>
		
## 4). 打开$WILDFLY_HOME/standalone/configuration/standalone.xml文件
		
### a. 先找到 datasources/drivers 节点, 在 datasources/drivers 节点内添加上一步配置的的驱动，内容如下：
				
        <drivers>
            ...
		<driver name="mysql" module="com.mysql.jdbc">
                	<driver-class>com.mysql.jdbc.Driver</driver-class>
                	<xa-datasource-class>com.mysql.jdbc.jdbc2.optional.MysqlXADataSource</xa-datasource-class>
		</driver>
        </drivers>
				
	注意: 此处 driver 节点 module attribute值为第3步骤 module 节点 name attribute 的值，它们必须相同
				
### b. 再在 drivers 的父节点 datasources 下添加一个 datasource 节点，内容如下：
		
	<datasources>
          ...
	    <datasource jndi-name="java:/EjbcaDS" pool-name="ejbcads" enabled="true" use-ccm="true" use-java-context="true">
              <connection-url>jdbc:mysql://127.0.0.1/ejbca?characterEncoding=UTF-8&amp;useSSL=false</connection-url>
              <driver>mysql</driver>
              <transaction-isolation>TRANSACTION_READ_COMMITTED</transaction-isolation>
              <pool>
                  <min-pool-size>5</min-pool-size>
                  <max-pool-size>150</max-pool-size>
                  <prefill>true</prefill>
              </pool>
              <security>
                  <user-name>root</user-name>
                  <password>root</password>
              </security>
              <validation>
                  <check-valid-connection-sql>select 1;</check-valid-connection-sql>
                  <validate-on-match>true</validate-on-match>
                  <background-validation>false</background-validation>
              </validation>
              <statement>
                  <prepared-statement-cache-size>50</prepared-statement-cache-size>
                  <share-prepared-statements>true</share-prepared-statements>
              </statement>
          </datasource>
          <drivers>
              ...
		<driver name="mysql" module="com.mysql.jdbc">
                  <driver-class>com.mysql.jdbc.Driver</driver-class>
                  <xa-datasource-class>com.mysql.jdbc.jdbc2.optional.MysqlXADataSource</xa-datasource-class>
              	</driver>
          </drivers>
      </datasources>
			
	i: connection-url 填写你实际的数据库连接
	ii: driver 填写 1)配置的 driver name attribute 值
	iii: jndi-name 为 ejbca 里面配置的 jndi-name名字，必须一致
			
	同时在 mysql 创建 ejbca 名的数据库
		
# 5. 修改$WILDFLY_HOME/bin/standalone.conf文件

    	JAVA_OPTS="-Xms2048m -Xmx2048m -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256m -Djava.net.preferIPv4Stack=true"
	  
    	把 -Xmx -Xms 增加到 2048以上，且值相同， -XX:MetaspaceSize 修改为256M or 512M, 且 -XX:MetaspaceSize and -XX:MaxMetaspaceSize 值要相同
	
# 6. 启动 wildfly
	
   	$WILDFLY_HOME/bin/standalone.sh

# 7. 用 jboss-cli 连接 wildfly
	
    	$WILDFLY_HOME/bin/jboss_cli.sh -c
    	等连接上以后，执行以下命令：
	
	/subsystem=remoting/http-connector=http-remoting-connector:remove
	/subsystem=remoting/http-connector=http-remoting-connector:add(connector-ref="remoting",security-realm="ApplicationRealm")
	/socket-binding-group=standard-sockets/socket-binding=remoting:add(port="4447")
	/subsystem=undertow/server=default-server/http-listener=remoting:add(socket-binding=remoting)
	:reload
	
	/subsystem=logging/logger=org.ejbca:add
	/subsystem=logging/logger=org.ejbca:write-attribute(name=level, value=DEBUG)
	/subsystem=logging/logger=org.cesecore:add
	/subsystem=logging/logger=org.cesecore:write-attribute(name=level, value=DEBUG)
	
# 8. 配置 EJBCA

	cd $EJBCA_HOME/conf 
	cp cesecore.properties.sample cesecore.properties
	cp crlstore.properties.sample crlstore.properties
	cp database.properties.sample database.properties
	cp ejbca.properties.sample ejbca.properties
	cp install.properties.sample install.properties
	cp web.properties.sample web.properties
	
	crlstore.properties 
		crlstor.enable=true
		crlstore.contextroot=your directory
		
	database.properties
		database.name=mysql
		database.url=jdbc:mysql://127.0.0.1:3306/ejbca?characterEncoding=UTF-8
		database.driver==com.mysql.jdbc.Driver
		database.username=database username
		database.password=database password
		
	ejbca.properties
		appserver.home=${env.APPSRV_HOME}
				此值对应第2步骤配置的 APPSRV_HOME  环境变量
		ca.cmskeystorepass=your keystore password
		ejbca.cli.defaultusername=ejbca
		ejbca.cli.defaultpassword=ejbca

	install.properties
		ca.name=your ca name
		    for example: DemoManagmentCA
			
		ca.dn=your ca.dn
		    for example: DemohuanManagementCA,O=Demo EJBCA,C=SE
		
		ca.tokentype=soft
		ca.tokenpassword=your token password
		ca.keyspec=2048
		ca.keytype=RSA
		ca.signaturealgorithm=SHA256WithRSA
		ca.validity=7300
		ca.policy=null
		
	web.properties
		java.trustpassword=your trust password
		superadmin.cn=your superadmin cn	
		    for example:DemoSuperAdmin
		
		superadmin.password=your super admin password
		superadmin.batch=true
		httpsserver.password=your https server password
		httpsserver.hostname=ejbca install server ip or host
		httpsserver.dn=CN=${httpsserver.hostname},O=Demo EJBCA,C=SE

	其它没有说明的值都是默认值
  	因为各种密码比较多，最好所有密码设置成一样，这样免得搞混了，出现不必要的错误

# 9. 安装 EJBCA 
	在EJBCA_HOME目录
		1). ant clean deployear
			这一命令会多花一些时间，且要等 wildfly console 显示象下面的日志再执行一下命令：
			10:57:54,379 INFO  [org.wildfly.extension.undertow] (ServerService Thread Pool -- 71) WFLYUT0021: Registered web context: /ejbca/ra
			10:57:54,402 INFO  [org.wildfly.extension.undertow] (ServerService Thread Pool -- 73) WFLYUT0021: Registered web context: /ejbca/adminweb
			10:57:54,431 INFO  [org.jboss.as.server] (DeploymentScanner-threads - 2) WFLYSRV0010: Deployed "ejbca.ear" (runtime-name : "ejbca.ear")
				
		2). ant runinstall
			
		3). ant deploy-keystore
		
# 10. 再次用 jboss_cli.sh -c 连接上 wildfly

	 依次执行下面的命令：
	 
	(移除wildfly中已有的TLS and HTTP功能)
	/subsystem=undertow/server=default-server/http-listener=default:remove
	/subsystem=undertow/server=default-server/https-listener=https:remove
	/socket-binding-group=standard-sockets/socket-binding=http:remove
	/socket-binding-group=standard-sockets/socket-binding=https:remove
		
		
	(配置新的TLS)
	/interface=http:add(inet-address="0.0.0.0")
	/interface=httpspub:add(inet-address="0.0.0.0")
	/interface=httpspriv:add(inet-address="0.0.0.0")
	:reload
		
	/socket-binding-group=standard-sockets/socket-binding=http:add(port="8080",interface="http")
	/subsystem=undertow/server=default-server/http-listener=http:add(socket-binding=http)
	/subsystem=undertow/server=default-server/http-listener=http:write-attribute(name=redirect-socket, value="httpspriv")
	:reload
	(上面的命令执行后， wildfly console会有一些异常日志出现，不需要关心它们)
		
	/core-service=management/security-realm=SSLRealm:add()
	/core-service=management/security-realm=SSLRealm/server-identity=ssl:add(keystore-path="${jboss.server.config.dir}/keystore/keystore.jks", keystore-password="aerozh2018", alias="192.168.8.110")
	/core-service=management/security-realm=SSLRealm/authentication=truststore:add(keystore-path="${jboss.server.config.dir}/keystore/truststore.jks", keystore-password="aerozh2018")
	/socket-binding-group=standard-sockets/socket-binding=httpspriv:add(port="8443",interface="httpspriv")
	/socket-binding-group=standard-sockets/socket-binding=httpspub:add(port="8442", interface="httpspub")
		
	注意：上面第一个 keystore-password必须和 $EJBCA_HOME/conf/web.properties  httpsserver.password 相同
	      alias 必须要和  $EJBCA_HOME/conf/web.properties httpsserver.hostname 相同
	      第二个 keystore-password 必须和 $EJBCA_HOME/conf/web.properties  httpsserver.password 相同
        
	最好密码是一样的，这里很容易出错
	
# 11. 退出 jboss_cli.sh

	停止 wildfly 然后再启动它 (谨记：<b>必须要重启</b>)
	
# 12. 配置 wildfly undertow 服务
	
	再进用 jboss-cli.sh -c 连接 wildfly
	执行下面命令：
	
	/subsystem=undertow/server=default-server/https-listener=httpspriv:add(socket-binding=httpspriv, security-realm="SSLRealm", verify-client=REQUIRED)
	/subsystem=undertow/server=default-server/https-listener=httpspriv:write-attribute(name=max-parameters, value="2048")
	/subsystem=undertow/server=default-server/https-listener=httpspub:add(socket-binding=httpspub, security-realm="SSLRealm")
	/subsystem=undertow/server=default-server/https-listener=httpspub:write-attribute(name=max-parameters, value="2048")
	:reload

	/subsystem=undertow/server=default-server/https-listener=httpspriv:add(socket-binding=httpspriv, security-realm="SSLRealm", verify-client=REQUIRED)
	/subsystem=undertow/server=default-server/https-listener=httpspriv:write-attribute(name=max-parameters, value="2048")
	/subsystem=undertow/server=default-server/https-listener=httpspub:add(socket-binding=httpspub, security-realm="SSLRealm")
	/subsystem=undertow/server=default-server/https-listener=httpspub:write-attribute(name=max-parameters, value="2048")
	:reload

	/subsystem=undertow/server=default-server/ajp-listener=ajp-listener:add(socket-binding=ajp, scheme=https, enabled=true)
	:reload
		
# 13. 浏览器导入 superadmin.p12 证书， 证书地址 $EJBCA_HOME/p12/superadmin.p12
	
	EJBCA 管理后台：
		https://ejbca-server-ip(host):8443/ejbca/adminweb
		
	EJBCA 前台：
		http://ejbca-server-ip(host):8080/ejbca
		


# 14. wildfly console 类似下面的 ERROR 日志都可以忽略：
	11:21:25,129 ERROR [org.jboss.as.jsf] (MSC service thread 1-1) WFLYJSF0002: Could not load JSF managed bean class: org.ejbca.ui.web.admin.peerconnector.PeerConnectorsMBean
	11:21:25,252 ERROR [org.jboss.as.jsf] (MSC service thread 1-1) WFLYJSF0002: Could not load JSF managed bean class: org.ejbca.ui.web.admin.peerconnector.PeerMgmtMBean
	11:21:25,318 ERROR [org.jboss.as.jsf] (MSC service thread 1-1) WFLYJSF0002: Could not load JSF managed bean class: org.ejbca.ui.web.admin.peerconnector.PeerConnectorMBean

