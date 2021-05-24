---
title:      " Nginx配置location中proxy_pass的'/'号的作用 "
date:       2020-05-26
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Nginx ]

---

**真实案例，就因为在配置时，少写了一个字符“/”，就造成访问不通报错，因而接到投诉。那么是怎么引起的呢？原因就是：Nginx在配置proxy_pass代理转接时，少些“/”字符造成的。有同学就有疑问，加不加“/”,区别真的那么大吗？我们带着这个疑问，来探究下这个问题。**

# location目录匹配详解

nginx每个location都是一个匹配目录，nginx的策略是：访问请求来时，会对访问地址进行解析，从上到下逐个匹配，匹配上就执行对应location大括号中的策略，并根据策略对请求作出相应。

依访问地址：http://www.example.com/book/index.html 为例，nginx配置如下：

```
location /book/  {                    
		proxy_connect_timeout 18000; ##修改成半个小时                        
		proxy_send_timeout 18000;                    
		proxy_read_timeout 18000;                    
		proxy_pass http://127.0.0.1:8080;        
}
```

那访问时就会匹配这个location,从而把请求代理转发到本机的8080Tomcat服务中，Tomcat相应后，信息原路返回。总结：**location如果没有“/”时，请求就可以模糊匹配以字符串开头的所有字符串，而有“/”时，只能精确匹配字符本身。**

下面举个例子说明：

 配置location /book可以匹配/bookdada请求，也可以匹配/book\*/dada等等，只要以book开头的目录都可以匹配到。而location /book/必须精确匹配/book/这个目录的请求,不能匹配/bookdada/或/book\*/dada等请求。

# proxy_pass有无“/”的四种区别探究

访问地址都是以：http://www.book.com/bddd/index.html 为例。请求都匹配目录/bddd/

## 第一种：加"/"

```
location  /bddd/ {    
		proxy_pass  http://127.0.0.1:8080/;
}
```

测试结果，请求被代理跳转到：http://127.0.0.1:8080/index.html

## 第二种: 不加"/"

```
location  /bddd/ {            
		proxy_pass http://127.0.0.1:8080;
}
```

测试结果，请求被代理跳转到：http://127.0.0.1:8080/bddd/index.html



3# 第三种: 增加目录加"/"

```
location  /bddd/ {            
		proxy_pass http://127.0.0.1:8080/sun/;
}
```

测试结果，请求被代理跳转到：http://127.0.0.1:8080/sun/index.html

## 第四种：增加目录不加"/"

```
location  /bddd/ {    
		proxy_pass http://127.0.0.1:8080/sun;
}
```

测试结果，请求被代理跳转到：http://127.0.0.1:8080/sunindex.html

**总结**

location目录后加"/",只能匹配目录，不加“/”不仅可以匹配目录还对目录进行模糊匹配。而proxy_pass无论加不加“/”,代理跳转地址都直接拼接。

为了加深大家印象可以用下面的配置实验测试下：

```
server {   
  listen       80;   
  server_name  localhost;   # http://localhost/bddd01/xxx -> http://localhost:8080/bddd01/xxx

  location /bddd01/ {           
    proxy_pass http://localhost:8080;   
  }

  # http://localhost/bddd02/xxx -> http://localhost:8080/xxx   
  location /bddd02/ {           
    proxy_pass http://localhost:8080/;    
  }

  # http://localhost/bddd03/xxx -> http://localhost:8080/bddd03*/xxx   
  location /bddd03 {           
    proxy_pass http://localhost:8080;   
  }
  
  # http://localhost/bddd04/xxx -> http://localhost:8080//xxx，请注意这里的双斜线，好好分析一下。
  location /bddd04 {           
    proxy_pass http://localhost:8080/;   
  }

  # http://localhost/bddd05/xxx -> http://localhost:8080/hahaxxx，请注意这里的haha和xxx之间没有斜杠，分析一下原因。
  location /bddd05/ {           
    proxy_pass http://localhost:8080/haha;    
  }

  # http://localhost/bddd06/xxx -> http://localhost:8080/haha/xxx   
  location /bddd06/ {           
    proxy_pass http://localhost:8080/haha/;   
  }

  # http://localhost/bddd07/xxx -> http://localhost:8080/haha/xxx   
  location /bddd07 {           
    proxy_pass http://localhost:8080/haha;   
  } 
  # http://localhost/bddd08/xxx -> http://localhost:8080/haha//xxx，请注意这里的双斜杠。
  location /bddd08 {           
    proxy_pass http://localhost:8080/haha/;   
  }
}
```