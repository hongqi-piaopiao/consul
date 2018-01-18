#Consul
启动Consul服务

```
# Terminal
dk run -d -p 8500:8500 -p 8300:8300 -p 8301:8301 -p 8302:8302 -p 8600:8600 consul:0.9.2 agent -dev -bind=0.0.0.0 -client=0.0.0.0
```


[点击此处查看Consul服务](http://127.0.0.1:8500) 
> 正确展示 Consul UI

- - -


## 服务注册

修改 _build.gradle_ , 添加依赖

```
ext {
    springCloudVersion = 'Dalston.SR4'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-starter-parent:${springCloudVersion}"
    }
}

dependencies {
	...
	compile('org.springframework.boot:spring-boot-starter-actuator')
	compile('org.springframework.cloud:spring-cloud-starter-consul-discovery')
	...
}


```

配置 _bootstrap.yml_

```
spring:
  cloud:
    consul:
      enabled: true
      hostname: localhost
      port: 8500
      ribbon:
        enabled: true
      discovery:
        enabled: true
        register: true

```

修改 _MstUserServiceApplication_ 

```
@SpringBootApplication
@EnableDiscoveryClient
public class MstUserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(MstUserServiceApplication.class, args);
    }
}

```


[点击此处查看服务注册](http://127.0.0.1:8500/ui/#/dc1/services)
> 能在Consul里面看到服务已经注册，但是健康检查不过

## Health Check

[点击此处检查当前服务健康状态](http://127.0.0.1:8090/health)
> 看到未授权，拿到401


修改 _WebSecurityConfig_ ,让 `/health` 跳过权限认证

```Java
http.csrf().disable()
           .authorizeRequests()
           .antMatchers(HttpMethod.GET, "/health").permitAll()
           .antMatchers(HttpMethod.POST, "/api/authentication").permitAll()
           .anyRequest().authenticated();
```

重启服务<br>

[点击此处再次检查当前服务健康状态](http://127.0.0.1:8090/health)
> 看到 {"description":"Spring Cloud Consul Discovery Client","status":"UP"}

[点击此处查看UserService在Consule里面已经正常](http://127.0.0.1:8500/ui/#/dc1/services/mst-users-service)
> consul 里面 user service 已经变成绿色的



## 服务发现

### 服务之间的服务发现 -- FeignClient

#### 在UsersService

添加一个`/api/users/names`并**跳过权限认证**的API

#### 在GoodsService

修改 _build.gradle_ , 添加依赖

```
dependencies {
    ...
    compile('org.springframework.cloud:spring-cloud-starter-feign')
    ...
}
```
修改 _MstGoodsServiceApplication_

```
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class MstGoodsServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(MstGoodsServiceApplication.class, args);
    }
}
```

创建 UserClient 

```
@FeignClient("mst-users-service")
public interface UserClient {

    @GetMapping("/api/users/names")
    List<String> getUserNames();

}
```

重启服务，发送


###客户端发现服务 -- ConsuleTemplate + Nginx

####安装

```
brew install consul-template
brew install nginx
```

查看Nginx服务

```
brew services list
```

>nginx  started username /Users/username/Library/LaunchAgents/homebrew.mxcl.nginx.plist

如果Nginx没有起来请执行

```
brew services restart nginx
```

设置ConsulTemplate

```
mkdir consule-template
cd consule-template 
```


生产ConsulTemplate Config文件

```

cat >> ./config.json << EOF
{
  "consul": {
    "address": "http://127.0.0.1:8500"
  },
  "template": {
    "source": "./config.ctmpl",
    "destination": "/usr/local/etc/nginx/servers/ms-nginx.conf",
    "command": "brew services reload nginx"
  }
}
EOF

```

生成 Nginx Conf 模板

```
cat >> ./config.ctmpl << EOF
{{range services}} {{$name := .Name}} {{$service := service .Name}}
upstream {{$name}} {
  #zone upstream-{{$name}} 64k;
  {{range $service}}server {{.Address}}:{{.Port}} max_fails=3 fail_timeout=60 weight=1;
  {{else}}server 127.0.0.1:65535; # force a 502{{end}}
} {{end}}

server {
  listen 8999 default_server;
  
{{range services}} {{$name := .Name}}
  location ~* {{ $name }} {
    proxy_pass http://{{$name}};
  }
{{end}}

  location / {
    root /usr/share/nginx/html/;
    index index.html;
  }
}
EOF
```

启动ConsulTemplate

```
consul-template -config ./config.json
```

查看生成好的模板

```
vim /usr/local/etc/nginx/servers/ms-nginx.conf
```

[点此查看Consule Key/Value](http://127.0.0.1:8500/ui/#/dc1/kv/)


添加 Key/Value


`matchers/mst-users-service` -> `/api/(users)`

`matchers/mst-goods-service` -> `/api/(goods)`

修改 _config.ctmpl_

```
...
{{range services}} {{$name := .Name}}
  {{if (printf "matchers/%s" $name | keyExists)}}
  location ~* {{ printf "matchers/%s" $name | key }} {
    proxy_pass http://{{$name}};
  }
  {{end}}
{{end}}
...
```

重启 ConsuleTemplate

查看生成好的模板

```
vim /usr/local/etc/nginx/servers/ms-nginx.conf
```

**修改 Key/Value 的值，查看模板的改变**

**添加或停掉几个服务，查看模板的改变**

- - -

#That's all, Thanks






