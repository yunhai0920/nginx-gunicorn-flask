#myproject


### python web 部署

web开发中，各种语言争奇斗艳，web的部署方面，却没有太多的方式。简单而已，大概都是 nginx 做前端代理，中间 webservice 调用程序脚本。大概方式：`nginx + webservice + script`

nginx 不用多说，一个高性能的web服务器。通常用来在前端做反向代理服务器。所谓正向与反向（reverse），只是英文说法翻译。代理服务，简而言之，一个请求经过代理服务器从局域网发出，然后到达互联网上服务器，这个过程的代理为正向代理。如果一个请求，从互联网过来，先进入代理服务器，再由代理服务器转发给局域网的目标服务器，这个时候，代理服务器为反向代理（相对正向而言）。

> 正向代理：{ 客户端 ---》 代理服务器 } ---》 服务器 

> 反向代理：客户端 ---》 { 代理服务器 ---》 服务器 } 
>  
> {} 表示局域网

nginx既可以做正向，也可以做反向。
webservice 的方式同样也有很多方式。常见的有`FastCGI`，`WSGI`等。我们采用` gunicorn`为 wsgi容器。python为服务器script，采用`flask`框架。同时采用supervisor管理服务器进程。也就是最终的部署方式为：
`nginx + gunicorn + flask ++ supervisor`

### 创建一个项目
   
    mkdir myproject
 
### 创建 python 虚拟环境
virtualenv 可以说是 python 的一个大杀器。用来在一个系统中创建不同的 python 隔离环境。相互之间还不会影响，使用简单到令人发指。这里我用的是anaconda2包中conda
命令来安装虚拟环境的

    cd myproject
    conda create ***（环境名）python=***(python版本号)

创建了 venv 环境之后，激活就可以了
    
    source activate ***(环境名)

### 安装 python web 框架 ---flask

flask 是一个 python web micro framework。简洁高效，使用也很简单。flask 依赖两个库 werkzeug 和 jinjia2。采用 pip 方式安装即可。
    
    conda install flask

测试我们的 flask 安装是否成功，并使用 flask 写一个简单的 web 服务。
    vim myapp.py
     
    from flask import Flask
    app = Flask(__name__)
    @app.route('/')
    def index():
        return 'hello world'

启动 flask
    python myapp.py
此时，用浏览器访问 http://127.0.0.1:5000 就能看到网页显示 `hello world`。

### 使用 gunicorn 部署 python web 
    
现在我们使用 flask 自带的服务器，完成了 web 服务的启动。生产环境下，flask 自带的 服务器，无法满足性能要求。我们这里采用 gunicorn 做 wsgi容器，用来部署 python。

### 安装 gunicorn
   
     conda install gunicorn

pip 是一个重要的工具，python 用来管理包。还有一个最佳生产就是每次使用 pip 安装的库，都写入一个 requirement 文件里面，既能知道自己安装了什么库，也方便别人部署时，安装相应的库。
     
     pip freeze > requirements.txt

以后每次 pip 安装了新的库的时候，都需freeze 一次。

当我们安装好 gunicorn 之后，需要用 gunicorn 启动 flask，注意 flask 里面的__name__里面的代码启动了 app.run(),这个含义是用 flask 自带的服务器启动 app。这里我们使用了 gunicorn，myapp.py  就等同于一个库文件，被 gunicorn 调用。

     gunicron -w4 -b0.0.0.0:8000 myapp:app

此时，我们需要用 8000 的端口进行访问，原先的5000并没有启用。其中 gunicorn 的部署中，，-w 表示开启多少个 worker，-b 表示 gunicorn 开发的访问地址。
 
想要结束 gunicorn 只需执行 pkill gunicorn，有时候还的 ps 找到 pid 进程号才能 kill。可是这对于一个开发来说，太过于繁琐，因此出现了另外一个神器---`supervisor`，一个专门用来管理进程的工具，还可以管理系统的工具进程。

### 安装 supervisor
    conda install supervisor
    echo_supervisord_conf > supervisor.conf   # 生成 supervisor 默认配置文件
    vim supervisor.conf                       # 修改 supervisor 配置文件，添加 gunicorn 进程管理

在myapp supervisor.conf   配置文件底部添加  (注意我的工作路径是` /home/rsj217/rsj217/`)

    [program:myapp]
    command=/home/hj/anaconda2/envs/***(虚拟环境名)/bin/gunicorn -w4 -b0.0.0.0:8000 myapp:app    ; supervisor启动命令
    directory=/home/hj/myproject                                                 ; 项目的文件夹路径
    startsecs=0                                                                             ; 启动时间
    stopwaitsecs=0                                                                          ; 终止等待时间
    autostart=false                                                                         ; 是否自动启动
    autorestart=false                                                                       ; 是否自动重启
    stdout_logfile=/home/hj/myproject/log/gunicorn.log                           ; log 日志
    stderr_logfile=/home/hj/myproject/log/gunicorn.err                           ; 错误日志
    
supervisor的基本使用命令

    supervisord -c supervisor.conf                             通过配置文件启动supervisor
    supervisorctl -c supervisor.conf status                    察看supervisor的状态
    supervisorctl -c supervisor.conf reload                    重新载入 配置文件
    supervisorctl -c supervisor.conf start [all]|[appname]     启动指定/所有 supervisor管理的程序进程
    supervisorctl -c supervisor.conf stop [all]|[appname]      关闭指定/所有 supervisor管理的程序进程

supervisor 还有一个web的管理界面，可以激活。更改下配置

    [inet_http_server]         ; inet (TCP) server disabled by default
    port=127.0.0.1:9001        ; (ip_address:port specifier, *:port for all iface)
    username=user              ; (default is no username (open server))
    password=123               ; (default is no password (open server))

    [supervisorctl]
    serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL  for a unix socket
    serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
    username=user              ; should be same as http_username if set
    password=123                ; should be same as http_password if set
    ;prompt=mysupervisor         ; cmd line prompt (default "supervisor")
    ;history_file=~/.sc_history  ; use readline history if available

现在可以使用 supervsior 启动 gunicorn啦。运行命令 `supervisord -c supervisor.conf `

访问 http://127.0.0.1:9001 可以得到 supervisor的web管理界面，访问 http://127.0.0.1:2170 可以看见gunciron 启动的返回的 hello world

### 安装配置 nginx

采用 apt-get方式安装最简单。运行 `sudo apt-get install nginx`。安装好的nginx的二进制文件放在 `/usr/sbin/`文件夹下面。而nginx的配置文件放在 `/etc/nginx`下面。

使用 supervisor 来管理 nginx。这里需要注意一个问题，linux的权限问题。nginx是sudo的方式安装，启动的适合也是 root用户，那么我们现在也需要用 root用户启动supervisor。增加下面的配置文件

    [program:nginx]
    command=/usr/sbin/nginx
    startsecs=0
    stopwaitsecs=0
    autostart=false
    autorestart=false
    stdout_logfile=/home/hj/myproject/log/nginx.log
    stderr_logfile=/home/hj/myproject/log/nginx.err

到此为止，进步的 web 部属已经完成。当然，最终我们需要把项目代码部属到服务器上.批量的自动化部属需要另外一个神器 fabric.具体使用，就不再这篇笔记阐述。项目源码中包含了fabric文件。下载fabric，更改里面的用户名和秘密，就可以部属在自己或者远程的服务器上了。






  
        










