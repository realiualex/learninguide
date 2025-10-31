# Nginx 配置

## Nginx 转发规则

1. 在处理转发的时候，如果location后面 直接跟 /path，那么就会按path匹配，以及匹配/path后的其他地址。

2. 但如果使用 location = /path ，那么就精确到只匹配/path
3. 如果location 后面跟 ~ 那么再后面可以跟正则，并且可以用()进行分组，方便在location内部使用分组的结果进行处理。需要注意的是，此时是区分大小写的。比如下面的示例，就是将所有/zts/的请求，转发到目标地址的/
4. 如果location后面跟 ~* 那么是不区分大小写的正则

```
	location ~ ^/path/(.*) {
		set $backend BACKEND_URL;
		proxy_pass http://$backend/$1;
	}

```
5. 默认情况下location里的=优先级最高，其次是是最长prefix.

### 有关Nginx 转发原理

如果 proxy_pass 后面跟的是域名(可以包含端口)，但没有包含路径，注意，哪怕只有一个 / 也叫做路径，比如这个格式: http://127.0.0.1:8080; 而不是这个格式 http://127.0.0.1:8080/；那么location 后面的 /path/to/path，只要路径匹配，则将原来的路径追加到 proxy_pass 的地址后面。示例，访问 http://nginx/api 这个地址，则会被转发到 http://127.0.0.1:8080/api 这里，包括/apixxx 以及/api/xxx，以及/apixxx?id=xxx 都会转发

```
location /api {
	proxy_pass http://127.0.0.1:8080;
}
```

但是，如果proxy_pass 后面跟了路径，则 location 后面的路径会被 proxy_pass 地址的路径替换。示例：访问 http://nginx/api，实际访问地址是 http://127.0.0.1:8080/；如果访问的是 http://nginx/apixxxx ，则实际路径是 http://127.0.0.1/xxxx，如果访问的是 http://nginx/api/xxxx ，则实际路径是 http://127.0.0.1//xxxx。主要关注的是 /api 被替换成了 /。同时，这种方式，如果路径里带了? 参数，则不会转发到proxy_pass的地址上。

```
location /api {
	proxy_pass http://127.0.0.1:8080/;
}
```

### 指定 dns 解析并转发

在Nginx的配置中，可以在 http {} 区块中指定dns resolver，这样在nginx中配置的转发域名，将会用这个dns做解析

```
    resolver 10.0.0.2 valid=60s;
    resolver_timeout 3s;
```



### 有关 rewrite 修改路径

在 server {} 区块中，如果通过 set 设置了变量 (在设置变量的时候，会强制 dns 解析)，那么可以用 rewrite 模块，将 某一个路径，替换为另外一个路径，这里也支持正则，然后在 proxy_pass 的时候，指定请求转发的具体位置.比如下面的配置，则表示请求 http://nginx/gw1/hash/aa/bb/cc?dd=ee 转发到 https://BACKEND_URL_ADDRESS/aa/bb/cc?dd=ee

        location /gw1/svc1 {
          set $hash_backend BACKEND_URL_ADDRESS;
          rewrite /gw1/svc1/(.+)$ /$1 break;
          proxy_pass https://$hash_backend;
        }

注意上述 rewrite 后面的proxy_pass 的地址，是不包含路径的，实际情况是：即使包含了路径，那么由于用了rewrite模块，也会被忽略。



学习资料

https://xuexb.github.io/learn-nginx/example/proxy_pass.html#url-%E5%8F%AA%E6%98%AF-host



## Nginx upstream的路由

nginx upstream支持 hash, ip hash, least_conn 等负载均衡算法，如果想要让某一个业务的流量，固定到达后端，我们可以使用 sticky cookie的方式，这样客户端只要带上这个cookie，就能到达同一个后端。但是也可以通过自定义的header来做路由。一个简单示例如下:

```nginx
    upstream backend{
      hash $http_biz_team;
      server 127.0.0.1:8081;
      server 127.0.0.1:8082;
    }
    server {
        listen 80;
        location / {
        proxy_pass http://backend;
        }
    }
    server {
     listen 8081;
     location / {
     return 200 "server 8081";
     }
    }
    server {
     listen 8082;
     location / {
     return 200 "server 8082";
     }
    }
```

在上述使用中，通过 curl 的时候，带上header名字就可以。需要注意：在map里是http_开头，并且变量是下划线，但实际header是中横线

```shell
 curl -H "biz-team: b" http://127.0.0.1
```



如果是使用 k8s 的 nginx ingress（k8官方的那个，并非f5的），配置起来就更简单了

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-test-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/upstream-hash-by: $http_biz_team
spec:
  ingressClassName: nginx
  rules:
  - host: nginx-test.test.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-test-svc
            port:
              number: 80
```





## nginx map 变量

在nginx中，如果想要将某一个header存到变量里，比如 biz-team (注意是中横线)，那么在map里要用 $http_biz_team

```nginx
    map $http_biz_team $biz_team {
      default "";
      ~^(.*)$ $1;
    }
```

在 nginx 中通过判断访问的path，决定转发路径

```nginx
map $uri $is_api_match {
    default 0;
    ~^/api/v1/sol(_mainnet|_testnet)?/(query|update) 1;
    ~^/api/v1/polkadot(_mainnet|_testnet)?/(query|update) 1;
}

server {
        listen 80;
        server_name  _;
        client_max_body_size 8M;
        resolver 10.0.0.2 valid=60s;
        resolver_timeout 3s;
        location ~* ^/api/v1/ {
          if ($is_api_match) {
            set $backend aaa.com;
            proxy_pass http://$backend;
        }
            set $backend bbb;
            proxy_pass http://$backend;
    }
}
```

## Nginx proxy_pass保留域名

当nginx的proxy_pass要转发到后面多个域名的时候，如果访问的地址必须要求Host是一致的（比如做了SNI），但最前端域名又与upstream域名不一致。那么可以使用两层proxy_pass解决。为了不占用过多端口，在内层proxy_pass的监听中，可以监听一个unix socket.

```nginx
upstream lambda {
      server  unix:/tmp/nginx1.sock;
      server  unix:/tmp/nginx2.sock;
}
server {
        listen unix:/tmp/nginx1.sock;
        location / {
                set $backend abc.lambda-url.ap-northeast-1.on.aws;
                proxy_set_header Host      $backend;
                proxy_pass https://$backend;
        }
    }
server {
        listen unix:/tmp/nginx2.sock;
        location / {
                set $backend bcd.lambda-url.ap-northeast-1.on.aws;
                proxy_set_header Host      $backend;
            proxy_pass https://$backend;
        }
    }
server {
        listen 80;
        server_name 123.com;
        location / {
            proxy_pass http://lambda;
        }
    }
```


## nginx ip 限制
在启用了 ip 限制模块的情况下，我们可以在 location 里用 `allow` 和 `deny` 的关键字，对访问 nginx 的ip做限制，但有时候用户的请求并不是直接访问到 nginx，用户也是经过层层代理访问过来，此时我们可以看用户最原始的ip，是否在 x-forwarded-for 的header里，如果在，我们就可以将这个ip取出来 (即修改 remote_addr为这个 x-forwarded-for 的 ip)，然后再用 `allow` 或 `deny` 控制

有两个地方要配置，在 http block 里要设置用户的真实ip来源于哪个网段，这里也可以设置为 `0.0.0.0/0`。在 location block里，设置真正的白名单，比如只允许 10.111，禁止其他的.
```
http {
    real_ip_header X-Forwarded-For;
    set_real_ip_from 10.111.0.0/16;     # 用户真实的ip来源于哪里
    real_ip_recursive on;
}

location /api/v1/ {
  set $backend my.server.com;
  proxy_pass http://$backend;
  allow 10.111.0.0/16;
  deny all;
}
        
```