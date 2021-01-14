# docker-v2ray-websocket-tls-web-bbr

@[TOC](docker 部署 v2ray  websocket+tls+web+bbr)


#  1.配置v2ray客户端
## 一、在docker hub上拉取v2ray最新版镜像

```bash
docker pull v2ray/official:latest
```


## 二、v2ray配置
1、创建文件夹，把v2ray配置和日志文件夹创建到本地
```bash
mkdir /etc/v2ray
mkdir /var/log/v2ray
```
2、 编辑配置文件
```bash
vim /etc/v2ray/config.json
```

```bash
{
  "inbounds": [
    {
      "port": 10000,//自定义端口号
      "listen":"127.0.0.1",//只监听 127.0.0.1，避免除本机外的机器探测到开放了 10000 端口
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "*******************",这里需要自己生成GUID，在线生成网站：www.ofmonkey.com
            "alterId": 64
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
        "path": "/ray"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

## 三. 运行v2ray
```bash
docker run -d --name v2ray -v /etc/v2ray:/etc/v2ray  -v /var/log/v2ray:/var/log/v2ray -p 10000:10000 v2ray/official  v2ray -config=/etc/v2ray/config.json
```

# 2.注册域名（自己百度）

# 3.生成tls
```bash
~/.acme.sh/acme.sh --issue -d 自己的域名 --standalone -k ec-256

~/.acme.sh/acme.sh --installcert -d 自己的域名 --fullchainpath /etc/v2ray/v2ray.crt --keypath /etc/v2ray/v2ray.key --ecc
```
# 4.配置nginx
## 创建文件夹
```bash
mkdir /etc/nginx
mkdir /etc/nginx/certs
mkdir /etc/nginx/conf.d
mkdir /var/log/nginx
mkdir /var/log/nginx/v2ray
mkdir /usr/share/nginx
mkdir /usr/share/nginx/v2ray
```

## 编辑nginx配置
```bash
vim /etc/nginx/conf.d/v2ray.conf
```
```bash
server {
	listen 443 ssl;
	
	server_name 自己的域名;
	
	ssl_certificate "/etc/v2ray/v2ray.crt"; //生成的密钥位置
    ssl_certificate_key "/etc/v2ray/v2ray.key";
	ssl_session_timeout 5m;
	ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
	ssl_protocols TLSv1.1 TLSv1.2;
	ssl_prefer_server_ciphers on;
	
	root /usr/share/nginx/v2ray;
	location / {
		index  index.html;
	}
   
 	location /ray { # 与 V2Ray 配置中的 path 保持一致
		proxy_redirect off;
		proxy_pass http://宿主容器ip:10000; #这里输入v2ray监听的端口号;查看宿主容器的ip：docker inspect 容器名称
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
		proxy_set_header Host $host;
		# Show real IP in v2ray access.log
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}
```
## 运行nginx容器
```bash
docker run --name nginx -v /etc/v2ray:/etc/v2ray -v /etc/nginx/conf.d:/etc/nginx/conf.d -v /var/log/nginx/v2ray:/var/log/nginx/v2ray -v /usr/share/nginx/v2ray:/usr/share/nginx/v2ray -p 443:443 -p 80:80 -d nginx 
```

# 5.BBR加速
```bash
wget -N --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh && chmod +x bbr.sh && bash bbr.sh
```
# 6.排错
## 查看日志文件
```bash
tail -f /var/log/nginx/error.log
```
###  failed (104: Connection reset by peer) while reading response header from upstream
### 排查 proxy_pass 的地址 是否是宿主容器地址


### failed (111: Connection refused) while connecting to upstream
### 除了网上的其他办法，还有就是排查一下/etc/v2ray  /etc/nginx/conf.d  /var/log/nginx/v2ray  /usr/share/nginx/v2ray有没有挂载到容器里
