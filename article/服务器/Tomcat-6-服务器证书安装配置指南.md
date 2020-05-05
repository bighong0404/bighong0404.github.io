转自 http://www.itrus.cn/service_view_79.html

1.  安装JDK 安装Tomcat需要JDK支持。

JDK1.6默认只支持 SSLv3 和 TLSv1 两个版本的https协议，JDK 1.7 版本默认禁用SSLv3，并支持 TLSv1、TLSv1.1及TLSv1.2。Tomcat 6及以下版本在使用 JDK 1.6及以下版本的运行环境时，可能存在无法禁用 SSLv3的情况。此时建议您升级 Tomcat 及 JDK版本，或变更使用 APR模块来配置SSL证书，以确保安全方式安装及使用服务器证书。Java SE Development Kit (JDK) 下载地址：http://www.oracle.com/technetwork/java/javase/downloads/index.html

2.  生成keystore文件生成密钥库文件keystore.jks需要使用JDK的keytool工具。命令行进入JDK或JRE下的bin目录，运行keytool命令（示例中粗体部分为可自定义部分，可根据实际配置情况相应修改）。
keytool -genkey -alias  server -keyalg RSA -keysize 2048 -keystore D:\keystore.jks -storepass password -keypass password

![](//upload-images.jianshu.io/upload_images/4039130-dfa27f1692e9529f.png)

以上命令中，server为私钥别名(-alias)，生成的keystore.jks文件默认放在D盘根目录下。

3.  生成证书请求文件(CSR)
keytool -certreq -alias server -sigalg SHA256withRSA -file D:\certreq.csr -keystore D:\keystore.jks -keypass password -storepass password

![](//upload-images.jianshu.io/upload_images/4039130-15119ecefe575cfa.png)
请备份密钥库文件keystore.jks，并稍后提交证书请求文件certreq.csr，等待证书签发。密钥库文件keystore.jks丢失将导致证书不可用。

二、  导入服务器证书

1.  获取服务器证书 
将证书签发邮件中的从BEGIN到 END结束的服务器证书内容（包括“-----BEGIN CERTIFICATE-----”和“-----END CERTIFICATE-----”）粘贴到记事本等文本编辑器中，并修改文件扩展名，保存为server.cer文件

2.  获取CA证书

将证书签发邮件中的从BEGIN到 END结束的两张 CA证书 内容（包括“-----BEGIN CERTIFICATE-----”和“-----END CERTIFICATE-----”）分别粘贴到记事本等文本编辑器中，并修改文件扩展名，保存为ca1.ce r和 ca2.cer文件。

3.  查看keystore文件内容
进入JDK安装目录下的bin目录，运行keytool命令查询keystore文件信息。
keytool -list -keystore D:\keystore.jks -storepass password

![](//upload-images.jianshu.io/upload_images/4039130-e0b53a9e0a66f10f.png)
查询到PrivateKeyEntry（或KeyEntry）属性的私钥别名(alias)为server。记住该别名，在稍后导入服务器证书时需要用到（示例中粗体部分为可自定义部分，可根据实际配置情况相应修改）。
注意，导入证书时，一定要使用生成证书请求文件时生成的keystore.jks文件。keystore.jks文件丢失或生成新的keystore.jks文件，都将无法正确导入您的服务器证书。

4.  导入证书(如果只有一张CA证书，则只需要导入一张CA证书)
导入第一张中级CA证书
keytool -import -alias ca1 -keystore D:\keystore.jks -trustcacerts -storepass password -file D:\ca1.cer -noprompt
导入第二张中级CA证书
keytool -import -alias ca2 -keystore D:\keystore.jks -trustcacerts -storepass password -file D:\ca2.cer -noprompt

![](//upload-images.jianshu.io/upload_images/4039130-71b74e0cdf5fd4fc.png)
导入服务器证书
keytool -import -alias server -keystore D:\keystore.jks -trustcacerts -storepass password -keypass password -file D:\server.cer

![](//upload-images.jianshu.io/upload_images/4039130-65e48504b0aaab6c.png)
导入服务器证书时，服务器证书的别名必须和私钥别名一致。请留意导入中级CA证书和导入服务器证书时的提示信息，如果您在导入服务器证书时使用的别名与私钥别名不一致，将提示“认证已添加至keystore中”而不是应有的“认证回复已安装在keystore中”。
证书导入完成，运行keystool命令，再次查看keystore文件内容
keytool -list -keystore D:\keystore.jks -storepass password

![](//upload-images.jianshu.io/upload_images/4039130-45fba0c21b31fa54.png)

三、  安装服务器证书
1.  配置Tomcat
复制已正确导入认证回复的keystore.jks文件到Tomcat安装目录下的conf目录。打开conf目录下的server.xml文件，找到并修改以下内容
<!—
           <Connector protocol="org.apache.coyote.http11.Http11Protocol"
               port="8443" SSLEnabled="true"
               maxThreads="150" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS" />
    -->
修改为
<Connector protocol="org.apache.coyote.http11.Http11Protocol"
                  port="443" SSLEnabled="true"
                   maxThreads="150" scheme="https" secure="true"
               keystoreFile="conf\keystore.jks" keystorePass="password"
               clientAuth="false" sslProtocol="TLS"
 ciphers="TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA256"  />
默认的SSL访问端口号为443，如果使用其他端口号，则您需要使用https://yourdomain:port的方式来访问您的站点。


2.  访问测试
重启Tomcat，访问https://youdomain:port，测试证书的安装。

四、  服务器证书的备份及恢复

在您成功的安装和配置了服务器证书之后，请务必依据下面的操作流程，备份好您的服务器证书，以防证书丢失给您带来不便。
1.  服务器证书的备份
备份服务器证书密钥库文件keystore.jks文件即可完成服务器证书的备份操作。
2.  服务器证书的恢复
请参照服务器证书安装部分，将服务器证书密钥库keystore.jks文件恢复到您的服务器上，并修改配置文件，恢复服务器证书的应用。
