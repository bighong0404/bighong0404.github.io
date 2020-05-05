**一、生成证书请求** 您需要使用CSR生成工具来创建证书请求。  
 1.下载AutoCSR： http://www.itrus.cn/soft/autocsr.rar

2. 生成服务器证书私钥及证书请求 
 运行AutoCSR.bat文件，按照操作提示填写证书注册信息。  
 以下是示例信息： 
 通用名（域名）：    test.itrus.com.cn 
 组织名称：          iTrus China Co.,Ltd. 
 部门名称：          VTN Support 
 省市名称：          Beijing 
 市或区名：          Beijing 
  
 3.备份私钥并提交证书请求 
 在程序目录下，将生成Cert目录。请妥善保存该目录下证书私钥文件server.key，并将证书请求文件certreq.csr提交给天威诚信。

二、安装服务器证书

1.获取服务器证书文件 
将证书签发邮件中的包含服务器证书代码的文本复制出来（包括“-----BEGIN CERTIFICATE-----”和“-----END CERTIFICATE-----”）粘贴到记事本等文本编辑器中。 
为保障服务器证书在客户端的兼容性，服务器证书需要安装两张中级CA证书(不同品牌证书，可能只有一张中级证书)。 
在服务器证书代码文本结尾，回车换行不留空行，并分别粘贴两张中级CA证书代码（包括“-----BEGIN CERTIFICATE-----”和“-----END CERTIFICATE-----”，每串证书代码之间均需要使用回车换行不留空行），修改文件扩展名，保存包含三段证书代码的文本文件为server.pem文件(如果只有一张中级证书，则只需要粘贴一张中级证书代码与服务器证书代码即可，并回车换行)。 

2.安装服务器证书
复制server.key、server.pem文件到Nginx安装目录下的conf目录。
打开Nginx安装目录下conf目录中的nginx.conf文件 
找到 
```
  # HTTPS server 
  # 
  #server { 
  #    listen       443; 
  #    server_name  localhost; 
  #    ssl                  on; 
  #    ssl_certificate      cert.pem; 
  #    ssl_certificate_key  cert.key; 
  #    ssl_session_timeout  5m; 
  #    ssl_protocols  SSLv2 SSLv3 TLSv1; 
  #    ssl_ciphers  ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
  #    ssl_prefer_server_ciphers   on; 
  #    location / { 
  #        root   html; 
  #        index  index.html index.htm; 
  #    } 
  #} 
```
将其修改为 
```
 server {
        listen 443;
        server_name  localhost;    
        ssl          on;     
        ssl_certificate server.pem;     #指定服务器证书路径
        ssl_certificate_key  server.key;    #指定私钥证书路径 
        ssl_session_timeout  5m;     
        ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;  #指定SSL服务器端支持的协议版本 
        ssl_ciphers  ALL：!ADH：!EXPORT56：RC4+RSA：+HIGH：+MEDIUM：+LOW：+SSLv2：+EXP;  #指定加密算法 
        ssl_prefer_server_ciphers   on; #在使用SSLv3和TLS协议时指定服务器的加密算法要优先于客户端的加密算法 
        location / { 
            root   html; 
            index  index.html index.htm; 
        } 
    } 
```
保存退出，并重启Nginx。 
通过https方式访问您的站点，测试站点证书的安装配置。
 
三、备份服务器证书 

在您成功的安装和配置了服务器证书之后，请务必依据下面的操作流程，备份好您的服务器证书，以防证书丢失给您的系统应用带来不便。 
1.   服务器证书的备份 
备份服务器证书私钥文件server.key，以及服务器证书文件server.pem即可完成服务器证书的备份操作。
 
2.   服务器证书的恢复 
请参照服务器证书配置部分，将服务器证书密钥文件恢复到您的服务器上，并修改配置文件，恢复服务器证书的应用。



四、相关事项

1、证书颁发机构

推荐天威诚信，具体请见：http://www.itrus.com.cn

2、 参考文档

http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_prefer_server_ciphers
《ngx_http_ssl_module》

http://nginx.org/cn/docs/http/configuring_https_servers.html
《nginx配置HTTPS服务器》 

http://www.itrus.cn/html/fuwuyuzhichi/fuwuqizhengshuanzhuangpeizhizhinan
《服务器证书配置指南》 
