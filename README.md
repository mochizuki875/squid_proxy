# squidでProxyを立てる
本手順ではdockerホスト上にコンテナとしてsquidを用いたproxyをデプロイする。  
サービスポートはport=8080とする。  
コンテナ起動時にコンテナ-dockerホスト間のポートルーティングを設定するため、  
外部からproxyを利用する際はdockerホストIP(またはFQDN):8080を指定すればproxy経由のアクセスを実現できる。


## 事前準備
### ディレクトリ構成
~~~
squid
     -docker-compose.yml
     -proxy
           -Dockerfile
           -squid.conf
           -docker-compose.yml
           -proxy_log
~~~

### squidログ出力先の権限変更
※権限についてはたぶんもっと打倒な値があるはずだけどここではフル権限を付ける
~~~
# chmod 777 squid/proxy/proxy_log
~~~


## 資材
### squid/docker-compose.yml
~~~
version: '2'

services:
  proxy:
    build:
      context: ./proxy
      # Set Proxy
      # args:
      #   - HTTP_PROXY=${HTTP_PROXY}
      #   - HTTPS_PROXY=${HTTPS_PROXY}

    ports:
      - 8080:8080
    privileged: true
    volumes:
      - /root/work/docker/squid/proxy/proxy_log:/var/log/squid/
~~~

### squid/proxy/Dockerfile
~~~
FROM centos:7
RUN su -

# Set Proxy
# ARG HTTP_PROXY
# ARG HTTPS_PROXY

RUN yum -y update
RUN yum -y install squid

RUN yum clean all

COPY ./squid.conf /etc/squid/squid.conf

EXPOSE 8080

CMD /sbin/init
~~~


### squid/proxy/squid.conf
~~~
#
# Recommended minimum configuration:
#

# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src 10.0.0.0/8     # RFC1918 possible internal network
acl localnet src 172.16.0.0/12  # RFC1918 possible internal network
acl localnet src 192.168.0.0/16 # RFC1918 possible internal network
acl localnet src fc00::/7       # RFC 4193 local private network range
acl localnet src fe80::/10      # RFC 4291 link-local (directly plugged) machines

acl SSL_ports port 443
acl Safe_ports port 80          # http
acl Safe_ports port 21          # ftp
acl Safe_ports port 443         # https
acl Safe_ports port 70          # gopher
acl Safe_ports port 210         # wais
acl Safe_ports port 1025-65535  # unregistered ports
acl Safe_ports port 280         # http-mgmt
acl Safe_ports port 488         # gss-http
acl Safe_ports port 591         # filemaker
acl Safe_ports port 777         # multiling http
acl CONNECT method CONNECT

# 接続元として許可するIPを設定
acl allowaddress src 192.168.2.0/255.255.255.0

#
# Recommended minimum Access Permission configuration:
#
# Deny requests to certain unsafe ports
http_access deny !Safe_ports

# Deny CONNECT to other than secure SSL ports
http_access deny CONNECT !SSL_ports

# Only allow cachemgr access from localhost

# 設定したallowaddressからの接続許可設定①
http_access allow localhost manager allowaddress
http_access deny manager
# 設定したallowaddressからの接続許可設定②
http_access allow allowaddress

# We strongly recommend the following be uncommented to protect innocent
# web applications running on the proxy server who think the only
# one who can access services on "localhost" is a local user
#http_access deny to_localhost

#
# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
#

# Example rule allowing access from your local networks.
# Adapt localnet in the ACL section to list your (internal) IP networks
# from where browsing should be allowed
http_access allow localnet
http_access allow localhost

# And finally deny all other access to this proxy
http_access deny all

# Squid normally listens to port 3128
http_port 8080

# Uncomment and adjust the following to add a disk cache directory.
#cache_dir ufs /var/spool/squid 100 16 256

# Leave coredumps in the first cache dir
coredump_dir /var/spool/squid

#
# Add any of your own refresh_pattern entries above these.
#
refresh_pattern ^ftp:           1440    20%     10080
refresh_pattern ^gopher:        1440    0%      1440
refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
refresh_pattern .               0       20%     4320
~~~

### squid/proxy/docker-compose.yml
~~~
version: '2'

services:
  squid:
    image: proxy:latest
    ports:
      - 8080:8080
    privileged: true
    volumes:
      - /root/work/docker/squid/proxy/proxy_log:/var/log/squid/
~~~


## [方法1]Dockerfileからデプロイする手順

### コンテナデプロイ
~~~
# cd squid/proxy
# docker build -t proxy:latest ./
# docker run --privileged -d -p 8080:8080 -v /root/work/docker/squid/proxy/proxy_log:/var/log/squid/ --name proxy proxy:latest /sbin/init

~~~
### コンテナ内サービス起動
~~~
# docker exec -it proxy bash

# systemctl start squid
# systemctl enable squid
~~~


## [方法2]docker-composeでデプロイする手順
## 流れ
 - docker-composeでイメージビルド&コンテナ初回起動
 - コンテナ初期設定&イメージコミット
 - コンテナ再起動


## docker-composeでイメージビルド&コンテナ初回起動

### コンテナ初回起動
~~~
# cd squid
# docker-compose up -d --build
~~~

## コンテナ初期設定&イメージコミット
### コンテナ内でsquidを起動
~~~
# docker exec -it squid_proxy_1 bash

# systemctl start squid
# systemctl enable squid

# exit
~~~


### コンテナからイメージのコミット
~~~
# docker commit squid_proxy_1 proxy
~~~ 
## コンテナ再起動
### 旧コンテナの停止&削除
~~~
# cd squid
# docker-compose stop
# docker-compose rm
~~~

### コンテナの起動
~~~
# cd squid/proxy
# docker-compose up -d
~~~


## 動作確認
自サーバから適当なインターネットサイトにproxy経由でアクセス
~~~
# curl -x 127.0.0.1:8080 https://www.yahoo.co.jp/
~~~

