##Supervisor作用

基于 Python 的进程管理工具，只能运行在 Unix-Like 的系统上，也就是无法运行在 Windows 上。Supervisor 官方版目前只能运行在 Python 2.4 以上版本

##Supervisor安装

```
#安装
sudo pip install supervisor
#创建配置文件
echo_supervisord_conf > /etc/supervisord.conf
```

Supervisor 有两个主要的组成部分：

* supervisord ，运行 Supervisor 时会启动一个进程 supervisord，它负责启动所管理的进程，并将所管理的进程作为自己的子进程来启动，而且可以在所管理的进程出现崩溃时自动重启。
* supervisorctl ，是命令行管理工具，可以用来执行 stop、start、restart 等命令，来对这些子进程进行管理。

##配置文件

默认的配置文件是下面这样的，但是这里有个坑需要注意，supervisord.pid 以及 supervisor.sock 是放在 /tmp 目录下，但是 /tmp 目录是存放临时文件，里面的文件是会被 Linux 系统删除的，一旦这些文件丢失，就无法再通过 supervisorctl 来执行 restart 和 stop 命令了，将只会得到 unix:///tmp/supervisor.sock 不存在的错误 。

```
[unix_http_server]
;file=/tmp/supervisor.sock   ; (the path to the socket file)
;修改为 /var/run 目录，避免被系统删除
file=/var/run/supervisor.sock   ; (the path to the socket file)
;chmod=0700                 ; socket file mode (default 0700)
;chown=nobody:nogroup       ; socket file uid:gid owner
;username=user              ; (default is no username (open server))
;password=123               ; (default is no password (open server))

;[inet_http_server]         ; inet (TCP) server disabled by default
;port=127.0.0.1:9001        ; (ip_address:port specifier, *:port for ;all iface)
;username=user              ; (default is no username (open server))
;password=123               ; (default is no password (open server))
...

[supervisord]
;logfile=/tmp/supervisord.log ; (main log file;default $CWD/supervisord.log)
;修改为 /var/log 目录，避免被系统删除
logfile=/var/log/supervisor/supervisord.log ; (main log file;default $CWD/supervisord.log)
logfile_maxbytes=50MB        ; (max main logfile bytes b4 rotation;default 50MB)
logfile_backups=10           ; (num of main logfile rotation backups;default 10)
loglevel=info                ; (log level;default info; others: debug,warn,trace)
;pidfile=/tmp/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
;修改为 /var/run 目录，避免被系统删除
pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
...
;设置启动supervisord的用户，一般情况下不要轻易用root用户来启动，除非你真的确定要这么做
;user=chrism                 ; (default is current user, required if root)
...

[supervisorctl]
; 必须和'unix_http_server'里面的设定匹配
;serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL  for a unix socket
;修改为 /var/run 目录，避免被系统删除
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket
;serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
;username=chris              ; should be same as http_username if set
;password=123                ; should be same as http_password if set
...
```

##权限问题

设置好配置文件后，应先创建上述配置文件中新增的文件夹。如果指定了启动用户 user，这里以 oxygen 为例，那么应注意相关文件的权限问题，包括日志文件，否则会出现没有权限的错误。例如设置了启动用户 oxygen，然后启动 supervisord 出现错误

```
Error: Cannot open an HTTP server: socket.error reported errno.EACCES (13)
```

就是由于上面的配置文件中 /var/run 文件夹，没有授予启动 supervisord 的用户 oxygen 的写权限。/var/run 文件夹实际上是链接到 /run，因此我们修改 /run 的权限。

```
sudo chmod 777 /run
```

一般情况下，我们可以用 root 用户启动 supervisord 进程，然后在其所管理的进程中，再具体指定需要以那个用户启动这些进程。

##使用浏览器来管理

supervisor 同时提供了通过浏览器来管理进程的方法，只需要注释掉如下几行就可以了。

```
;[inet_http_server]         ; inet (TCP) server disabled by default
;port=127.0.0.1:9001        ; (ip_address:port specifier, *:port for ;all iface)
;username=user              ; (default is no username (open server))
;password=123               ; (default is no password (open server))

[supervisorctl]
...
;serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
;username=chris              ; should be same as http_username if set
;password=123                ; should be same as http_password if set
```

##使用 include
在配置文件的最后，有一个 [include] 的配置项，跟 Nginx 一样，可以 include 某个文件夹下的所有配置文件，这样我们就可以为每个进程或相关的几个进程的配置单独写成一个文件。

```
[include]
files = /etc/supervisord.d/*.ini
```

进程的配置样例

一个简单的例子如下

```
; 设置进程的名称，使用 supervisorctl 来管理进程时需要使用该进程名
[program:your_program_name] 
command=python server.py --port=9000
;numprocs=1                 ; 默认为1
;process_name=%(program_name)s   ; 默认为 %(program_name)s，即 [program:x] 中的 x
directory=/home/python/tornado_server ; 执行 command 之前，先切换到工作目录
user=oxygen                 ; 使用 oxygen 用户来启动该进程
; 程序崩溃时自动重启，重启次数是有限制的，默认为3次
autorestart=true            
redirect_stderr=true        ; 重定向输出的日志
stdout_logfile = /var/log/supervisord/tornado_server.log
loglevel=info
```

##设置日志级别

`loglevel` 指定了日志的级别，用 Python 的 print 语句输出的日志是不会被记录到日志文件中的，需要搭配 Python 的 logging 模块来输出有指定级别的日志。

##多个进程
按照官方文档的定义，一个 `[program:x]` 实际上是表示一组相同特征或同类的进程组，也就是说一个 `[program:x]` 可以启动 多个进程 。这组进程的成员是通过 `numprocs` 和 `process_name` 这两个参数来确定的，这句话什么意思呢，我们来看这个例子。

```
; 设置进程的名称，使用 supervisorctl 来管理进程时需要使用该进程名
[program:foo] 
; 可以在 command 这里用 python 表达式传递不同的参数给每个进程
command=python server.py --port=90%(process_num)02d
directory=/home/python/tornado_server ; 执行 command 之前，先切换到工作目录
; 若 numprocs 不为1，process_name 的表达式中一定要包含 process_num 来区分不同的进程
numprocs=2                   
process_name=%(program_name)s_%(process_num)02d; 
user=oxygen                 ; 使用 oxygen 用户来启动该进程
autorestart=true            ; 程序崩溃时自动重启
redirect_stderr=true        ; 重定向输出的日志
stdout_logfile = /var/log/supervisord/tornado_server.log
loglevel=info
```

上面这个例子会启动两个进程，`process_name` 分别为 `foo:foo_01` 和 `foo:foo_02` 。通过这样一种方式，就可以用一个 `[program:x]` 配置项，来启动一组非常类似的进程。

再介绍两个配置项 `stopasgroup` 和 `killasgroup`

```
; 默认为 false，如果设置为 true，当进程收到 stop 信号时，会自动将该信号发给该进程的子进程。如果这个配置项为 true，那么也隐含 killasgroup 为 true。例如在 Debug 模式使用 Flask 时，Flask 不会将接收到的 stop 信号也传递给它的子进程，因此就需要设置这个配置项。
stopasgroup=false             ; send stop signal to the UNIX process 

; 默认为 false，如果设置为 true，当进程收到 kill 信号时，会自动将该信号发给该进程的子进程。如果这个程序使用了 python 的 multiprocessing 时，就能自动停止它的子线程。
killasgroup=false             ; SIGKILL the UNIX process group (def false)
```

更详细的配置例子，可以参考如下，官方文档在 [这里](http://supervisord.org/configuration.html#program-x-section-settings)

```
;[program:theprogramname]
;command=/bin/cat              ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=999                  ; the relative start priority (default 999)
;autostart=true                ; start at supervisord start (default: true)
;autorestart=unexpected        ; whether/when to restart (default: unexpected)
;startsecs=1                   ; number of secs prog must stay running (def. 1)
;startretries=3                ; max # of serial start failures (default 3)
;exitcodes=0,2                 ; 'expected' exit codes for process (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
;killasgroup=false             ; SIGKILL the UNIX process group (def false)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (default 10)
;stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (default 10)
;stderr_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A="1",B="2"       ; process environment additions (def no adds)
;serverurl=AUTO                ; override serverurl computation (childutils)
```

##将多个进程按组管理

Supervisor 同时还提供了另外一种进程组的管理方式，通过这种方式，可以使用 supervisorctl 命令来管理一组进程。跟 [program:x] 的进程组不同的是，这里的进程是一个个的 [program:x] 。

```
[group:thegroupname]
programs=progname1,progname2  ; each refers to 'x' in [program:x] definitions
priority=999                  ; the relative start priority (default 999)

```

当添加了上述配置后， `progname1` 和 `progname2` 的进程名就会变成 `thegroupname:progname1` 和 `thegroupname:progname2` 以后就要用这个名字来管理进程了，而不是之前的 `progname1` 。

以后执行 `supervisorctl stop thegroupname:` 就能同时结束 `progname1` 和 `progname2` ，执行 `supervisorctl stop thegroupname:progname1` 就能结束 `progname1` 。`supervisorctl` 的命令我们稍后介绍。

##启动 supervisord
执行 supervisord 命令，将会启动 supervisord 进程，同时我们在配置文件中设置的进程也会相应启动。

```
# 使用默认的配置文件 /etc/supervisord.conf
supervisord
# 明确指定配置文件
supervisord -c /etc/supervisord.conf
# 使用 user 用户启动 supervisord
supervisord -u user
```

supervisorctl 命令介绍

* supervisorctl stop program_name， 停止某一个进程(program_name)，program_name 为 [program:program_name] 里配置的值
* supervisorctl start，program_name，启动某个进程
* supervisorctl restart ，program_name，重启某个进程
* supervisorctl stop groupworker: ，结束所有属于名为 groupworker 这个分组的进程(start，restart 同理)
* supervisorctl stop groupworker:name1 ，结束 groupworker:name1 这个进程(start，restart 同理)
* supervisorctl stop all，停止全部进程，注：start、restart、stop 都不会载入最新的配置文件。
* supervisorctl reload，载入最新的配置文件，停止原有进程并按新的配置启动、管理所有进程。
* supervisorctl update，根据最新的配置文件，启动新配置或有改动的进程，配置没有改动的进程不会受影响而重启。
* 注意：显示用 stop 停止掉的进程，用 reload 或者 update 都不会自动重启。

##开机自动启动 Supervisord
Supervisord 默认情况下并没有被安装成服务，它本身也是一个进程。官方已经给出了脚本可以将 Supervisord 安装成服务，可以参考 这里 查看各种操作系统的安装脚本，但是我用官方这里给的 Ubuntu 脚本却无法运行。

安装方法可以参考 [serverfault](http://serverfault.com/questions/96499/how-to-automatically-start-supervisord-on-linux-ubuntu) 上的回答。

比如我是 Ubuntu 系统，可以这么安装，这里选择了另外一个脚本

```
# 下载脚本
sudo su - root -c "sudo curl https://gist.githubusercontent.com/howthebodyworks/176149/raw/d60b505a585dda836fadecca8f6b03884153196b/supervisord.sh > /etc/init.d/supervisord"
# 设置该脚本为可以执行
sudo chmod +x /etc/init.d/supervisord
# 设置为开机自动运行
sudo update-rc.d supervisord defaults
# 试一下，是否工作正常
service supervisord stop
service supervisord start
```

注意：这个脚本下载下来后，还需检查一下与我们的配置是否相符合，比如默认的配置文件路径等，`pid` 文件路径等，如果存在不同则需要进行一些修改。

其实还有一个简单的方法，因为 `Linux` 在启动的时候会执行 `/etc/rc.local` 里面的脚本，所以只要在这里添加执行命令就可以

```
# 添加以下内容
/usr/local/bin/supervisord -c /etc/supervisord.conf
```

以上内容需要添加在 exit 命令前，而且由于在执行 rc.local 脚本时，PATH 环境变量未全部初始化，因此命令需要使用绝对路径。

