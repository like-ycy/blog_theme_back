---
title:      " Nginx location匹配规则 "
date:       2020-06-11
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Nginx ]

---

#一、location语法

```
location [=|~|~*|^~] uri { … }
```

其中，方括号中的四种标识符是可选项，用来改变请求字符串和uri的匹配方式。uri是待匹配的请求字符串，可以是不包含正则的字符串，这种模式被称为“标准的uri"；也可以包含正则，这种模式被称为"正则uri"。例如：

location ~ .*\.(php|php5)?$ {
　　root /var/www/html;
　　……
}

# 二、四种可选标识符

| 标识符 | 描述                                                         |
| ------ | ------------------------------------------------------------ |
| =      | **精确匹配：**用于标准uri前，要求请求字符串和uri严格匹配。如果匹配成功就停止匹配，立即执行该location里面的请求。 |
| ~      | **正则匹配：**用于正则uri前，表示uri里面包含正则，并且区分大小写。 |
| ~*     | **正则匹配：**用于正则uri前，表示uri里面包含正则，不区分大小写。 |
| ^~     | **非正则匹配；**用于标准uri前，nginx服务器匹配到前缀最多的uri后就结束，该模式匹配成功后，不会使用正则匹配。 |
| 无     | **普通匹配（最长字符匹配）；**与location顺序无关，是按照匹配的长短来取匹配结果。若完全匹配，就停止匹配。 |

# 三、匹配标识符案例

## 1. “=”精准匹配

```
location = /news/ {
            echo "test1";
        }
```

```
[root@test]# curl 192.168.233.22/news/
test1
```

## 2. "~"区分大小写正则匹配

```
location ~ \.(html) {
    echo 'test2';
}
location ~ \.(htmL) {
    echo 'test3';
}
```

```
[root@test]# curl 192.168.233.22/index.html
test2
[root@test]# curl 192.168.233.22/index.htmL
test3
```

## 3. “~\*”不区分大小写的正则匹配

```
location ~* \.(html){
            echo 'test4';
}
```

```
[root@test]# curl 192.168.233.22/index.htmL
test4
[root@test]# curl 192.168.233.22/index.html
test4
```

## 4. “^~”不进行正则匹配的标准匹配，只匹配前缀

```
location ^~ /index/ {
            echo 'test5';
}
```

```
[root@test]# curl 192.168.233.22/index/
test5
[root@test]# curl 192.168.233.22/index/heihei
test5
[root@test]# curl 192.168.233.22/index/asdnmkalsjd
test5
```

## 5. 普通匹配

```
location / {
            echo 'test6';
}
```

```
[root@test]# curl 192.168.233.22
test6
```

