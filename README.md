# 目的  
> 部署 pip private server    

# 使用范围  
> 内部私有 pip 源   

# 资源信息   

## os 需求

| 软件 |  版本 | 说明 |  
|-- | --| --|  
| python | 3.8.2 | 需编译  |  
| openssl | 1.1.1g | 需编译 |  
| centos | 7 或 8 |  |   
| nginx | 任意 | 用于请求代理, 通过 http status_code 获取下载信息 |   

## pip 需求

| pip |  版本 | 说明  | 
| -- | -- | -- |
| pam | any | 用于上传 pip 软件时候进行验证 |
| six | any | 依赖 |
| markupsafe | 2.0.1 |  |
| aws-sam-cli | 1.37.0 | |
| private-pypi | any } |
| pypiserver | any | |


# deploy    

## disable selinux 

```
/etc/selinux/config
SELINUX=disabled


init 6   <--- needed
```

## os dependence   

```
# yum install -y vim* 
# yum groupinstall -y "Development Tools"
```

## python-3.8 + openssl    

### source code compile    


### openssl compile    

```
# cd /usr/src
# tar xf openssl-1.1.1g.tar
# ./config --prefix=/usr/local/openssl
# make 
# make install

# echo "/usr/local/openssl/lib/" > /etc/ld.so.conf.d/openssl.conf
# ldconfig

```

### python compile    

```
# tar xf python3-el7.tar
# ./configure --with-openssl=/usr/local/openssl

```
###  验证 openssl install    

```
# strings /etc/ld.so.cache  | grep openssl
/usr/local/openssl/lib/libssl.so.1.1
/usr/local/openssl/lib/libssl.so

```
### python install 

```
# cd /usr/src/Python-3.8.3/
# make install
```
### verify python    

```
# /usr/local/python3/bin/python3 -V
Python 3.8.3

```

# private pip server install 

## add private pip source use to install pip package   

```
# cat /root/.pip/pip.conf
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=pypi.tuna.tsinghua.edu.cn

```

## install private pip service  

### install dependence package   

```
yum install -y libffi-devel.x86_64
```

## install private pip server program   

```
# /usr/local/python3/bin/pip3 install pam
# /usr/local/python3/bin/pip3 install six
# /usr/local/python3/bin/pip3 install  markupsafe==2.0.1
# /usr/local/python3/bin/pip3 install --user aws-sam-cli==1.37.0
# /usr/local/python3/bin/pip3 install private-pypi

.....
Building wheels for collected packages: pynacl
  Building wheel for pynacl (PEP 517) ... -            <- wait for build server 
.....

# /usr/local/python3/bin/pip3 install pypiserver

```

### program testing  

```
# /usr/local/python3/bin/private_pypi -h
SYNOPSIS
    private_pypi <command> <command_flags>

SUPPORTED COMMANDS
    server
    update_index
    github.init_pkg_repo
    github.gen_gh_pages
```

## install python virtualenv

```
# /usr/local/python3/bin/pip3 install  virtualenv

# export PRIVATE_PYPI=/apps/dat/firefly_pypi_private
# cd $PRIVATE_PYPI
#  /usr/local/python3/bin/virtualenv pypienv

# tree /apps/dat/firefly_pypi_private -L 2
/apps/dat/firefly_pypi_private
`-- pypienv
    |-- bin
    |-- lib
    `-- pyvenv.cfg

```


## private server config 

> virtual python = /apps/dat/firefly_pypi_private/pypienv   
> pypi pacakge root dir = /apps/dat/firefly_pypi_private/data    
> config file =   
> start_script = /apps/sh/firefly_pypi_private/start.py   
> start_service = systemctl restart vip-pypi    

```
# cat /apps/sh/firefly_pypi_private/start.py

#!/apps/dat/firefly_pypi_private/pypienv/bin/python
import pypiserver
from pypiserver import bottle
import pam
app = pypiserver.app(root='/apps/dat/firefly_pypi_private/data', auther=pam.authenticate)
bottle.run(app=app, host='127.0.0.1', port=8080, server='auto')

```


## start pypi server

```

# systemctl restart vip-pypi
# netstat -ntl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN
tcp6       0      0 :::22                   :::*                    LISTEN


```

# nginx proxy

```
vip-pypi.conf

server {
    listen       80;
    server_name  vip-pypi.vclound.com  vip-pypi.vip.vip.com;

    access_log /apps/logs/nginx/vip-pypi.vip.vip.access.log log_access;
    gzip on;
    gzip_proxied any;
    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
        proxy_pass http://127.0.0.1:8080;
        proxy_redirect off;
    }

     charset utf-8;
     error_page 404 /404.html;
     location = /40x.html {
     }

      error_page 500 502 503 504 /50x.html;
        location = /50x.html {
      }
      location = /vip-nginx-status {
          access_log off;

           allow 125.76.229.113;
           allow 60.195.252.106;
           allow 60.195.249.83;
           allow 10.0.0.0/8;
           allow 127.0.0.1;
           deny all;
           stub_status on;
       }
}


```


# pip upload 

## upload user 

> notice : pip upload user == centos7 user 
> use system pam module to authorized  

```
useradd -s /sbin/nologin  vip-pypi 
password vip-pypi
```

## upload config 

### config private upload profile 

```
cat ~/.pypirc


[distutils]
index-servers =
   local
 
[local]
repository: http://vip-pypi.vip.vip.com
username = vip-pypi
password = 上传密码
 
(最大限制上传文件为 500M)
```

###  upload pip require
```
/usr/local/python3/bin/pip install twine 
```
### upload howto example

```
/usr/local/python3/bin/twine  upload -r local /tmp/tf_nightly-2.6.0.dev20210505-cp36-cp36m-manylinux2010_x86_64.whl  --verbose
INFO     Using configuration from /root/.pypirc
Uploading distributions to http://vip-pypi.vip.vip.com
INFO     /tmp/tf_nightly-2.6.0.dev20210505-cp36-cp36m-manylinux2010_x86_64.whl (431.9 MB)
INFO     username set from config file
INFO     password set from config file
INFO     username: vip-pypi
INFO     password: <hidden>
Uploading tf_nightly-2.6.0.dev20210505-cp36-cp36m-manylinux2010_x86_64.whl
100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 452.9/452.9 MB • 00:06 • 71.3 MB/s
INFO     Response from http://vip-pypi.vip.vip.com/:
         200 OK
```

~

