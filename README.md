# koa-typescript-mongodb
- 此项目为一位vue前端开发,在做个人项目时对node后端学习并实践记录.技术栈主要为vue+koa+typescript+mongodb,并且有一些服务器配置,持续集成,代理,对象存储等等.此项目不包含具体代码展示,介绍的是工程搭建时学习步骤,可能需要的工具和可能遇到的问题.
- 本项目主要介绍在可以正常使用koa项目后所完成的一些操作,关于koa项目如果初始化,可自行进行查询搭建.并且对于框架的使用不作详细说明.

## 目录
* [基础](#基础)
* [mongodb安装](#mongodb安装)
* [本地koa连接mongodb](#本地koa连接mongodb)
* [koa相关工具组件](#koa相关工具组件)
  * koa-logger
  * koa-router
  * 错误处理
  * koa-body
  * koa-jwt + jsonwebtoken
* [CentOS 7.6 环境搭建](#CentOS 7.6 环境搭建)
  * 安装javajdk
  * 安装node
  * 安装mongodb
* [jenkins - 持续集成,自动化部署](#jenkins - 持续集成,自动化部署)
* [nginx相关操作](#nginx)

## 基础
1. 安装node, npm环境,在命令行node -v, npm -v 判断node, npm安装是否成功
2. 运行koa项目,打开地址可正常访问
 

## mongodb安装
1. windows 本地环境安装mongodb,前往官网安装mongodb,创建db(数据库)文件夹,logs(日志)文件夹
2. 进入mongodb的安装目下bin文件夹下,cmd执行`mongodb --dbpath db文件夹路径`,若出现没有结束状态,可以在此目录下重新打开一个cmd执行`mongo`若出现>_即mongodb本地启动成功.mongodb默认启动地址`mongodb://127.0.0.1:27017`


## 本地koa连接mongodb
1. 进入koa项目执行`npm i mongoose --save`
2. 连接数据库
```
src/config/db.ts

const mongoose = require('mongoose')
import config from './index'

const initDB = () => {
    mongoose.connect(
        config.dbPath,
        { 
            useNewUrlParser: true,
            useUnifiedTopology: true,
            useCreateIndex: true,
            useFindAndModify: false
        }
    )

    mongoose.connection.once('open', () => {
        console.log('connected to database;')
    })
}

export default initDB
```
在主入口文件引用重新启用项目,如果成功控制台会显示`connected to database;`
3. 相关目录
```
src/mondels // 模型层
src/controllers // 控制层
```


## koa相关工具组件
- koa-logger
  - 在控制台输出请求日志
  ```
    npm i koa-logger --save
    
    src/server.ts
    
    import logger from 'koa-logger'
    app.use(logger())
    
    app.use(async (ctx: { method: any; url: any }, next: () => any) => {
        const start = Number(new Date())
        await next()
        const end = Number(new Date())
        const ms = start - end
        console.log(`${ctx.method} ${ctx.url} - ${ms}ms`)
    })
  ```
- koa-router
  - 创建路由
  ```
    npm i koa-router --save
  
    src/routes/index.ts
    
    import Router from "koa-router";    // 导入koa-router
    import {index} from '../controllers/test'
    
    // 新建一个koa-router对象
    const router:Router = new Router({
        prefix: '/api' // 添加前缀
    });     
    
    router.get('/', index)
    
    export default router
  ```
- 错误处理
  - 监听错误并返回错误信息
  ```
    src/server.ts
    
    app.on('error', async (err, ctx) => {
        let errStruct = {
            errCode: 4000, // 可以自行定义错误码
            alert: err.message
        }
        if (err.status && err.status === 401) {
            errStruct.errCode = 4100
            errStruct.alert = '鉴权失败,请重新登录'
        }
    
        ctx.res.writeHead(200, {
            'content-Type': 'application/json'
        });
        ctx.res.end(JSON.stringify(errStruct));
    })
  ```
- koa-body
  - 处理请求内容和上传功能
  ```
    npm i koa-body --save
    
    src/server.ts
    app.use(koaBody({
        multipart: true, // 是否支持 multipart-formdate 的表单
        formidable: { // 配置更多的关于 multipart 的选项
            maxFieldsSize: 200 * 1024 * 1024
        }
    }))
  ```
  - 如果你使用了koa-bodyparser可能会存在冲突,二选一即可
- koa-jwt + jsonwebtoken
  - 控制登录路由权限验证,通过登录接口获取token存储在cookie中,在每次请求时携带在请求头中
  - 前端代码(axios请求拦截时添加)
  ```
    service.interceptors.request.use(
        config => {
            const token = CommonUtil.getCookie('token')
            config.headers['Authorization'] = "Bearer " + token // koa-jwt固定格式
            return config;
        },
        error => {
            return Promise.reject(error);
        }
    );
  ```
  - 后端代码
  ```
    npm i koa-jwt jsonwebtoken
    
    src/controllers/admin/login.ts
    
    const secret = '****' // 秘钥标识
    const token = jwt.sign({
        name: opts.username,
        _id: doc._id
    }, secret, { expiresIn: 60 * 60 * 24 } // 失效时长);
    ctx.body = {
        errCode: 0,
        data: {
            token: token,
            userId: doc._id
        },
        message: '登录成功' 
    }
    
    src/server.ts
    
    app.use(koajwt({
        secret: 秘钥标识
    }).unless({
        custom: ctx => { // 动态判断路由是否需要检测
            if (/\/admin\/register/.test(ctx.path) || /\/admin\/login/.test(ctx.path) || /^(\/weapp)/.test(ctx.path)) {
                return true
            } else {
                return false
            }
        }
        // path: [/\/admin\/register/, /\/admin\/login/, /^(\/weapp)/] // 固定不需要检测权限的路由
    }))
  ```

## CentOS 7.6 环境搭建
- 安装javajdk
  - [下载linux x64压缩包](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
  - 通过ftp工具上传到服务器
  - 将JDK压缩包  复制一份到/usr/local/src/作备份 `cp jdk-8u144-linux-x64.tar.gz /usr/local/src/`
  - 将jdk-8u144-linux-x64.tar.gz文件拷贝一份到/usr/java `cp jdk-8u144-linux-x64.tar.gz /usr/java`
  - `cd /usr/java` 在java目录下，解压JDK压缩文件 `tar -zxvf jdk-8u144-linux-x64.tar.gz`
  - 删除JDK压缩包 `rm -f jdk-8u144-linux-x64.tar.gz`
  - 编辑全局变量 `vim /etc/profile` 键盘按下i进入编辑模式 
  - 在文本最后一行添加如下之后保存退出(ESC -> :wq)
  ```
    #java environment
    export JAVA_HOME=/usr/java/jdk1.8.0_144 //是你的java安装包路径
    export CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
    export PATH=$PATH:${JAVA_HOME}/bin
  ```
  - 查看环境变量是否生效 -> `source /etc/profile` -> `java -version` 若出现版本号说明生效
- 安装node
  - [下载 node linux x64 压缩包](https://nodejs.org)
  - 可根据安装javajdk相关步骤进行解压
  - 编辑全局变量 `vim /etc/profile`
  ```
    #node environment
    export NODE_HOME=/usr/soft/node/node-v12.18.2-linux-x64
    export PATH=$PATH:$NODE_HOME/bin 
    export NODE_PATH=$NODE_HOME/lib/node_modules
  ```
  - 查看环境变量是否生效 -> `source /etc/profile` -> `node -v` 若出现版本号说明生效
- 安装mongodb
  - 可根据此网站进行安装(https://mirror.tuna.tsinghua.edu.cn/help/mongodb/)
  - 执行cat `/etc/mongod.conf` 查看相关配置
  - tips: 
    - 可能会出现因为没有创建db文件夹 或 端口号已被占用问题
    - 默认配置只允许本机连接.若需要远程连接服务器数据库,可以在mongod.conf(注释bindIp, 添加bindIpAll,也可指定ip连接)
    ```
    net:
      port: 27017
    # bindIp: 127.0.0.1  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses   or, alternatively, use the net.bindIpAll setting.
      bindIpAll: true	
    ```
    - 配置完,服务器需要对数据库启动的端口号设置完全组.

## jenkins - 持续集成,自动化部署
- `wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat/jenkins.repo` 添加jenkins源
- `rpm --import https://pkg.jenkins.io/redhat/jenkins.io.key` 安装jenkins key
- `sudo yum install jenkins` 安装jenkins  
- `systemctl start jenkins.service` 若报错 可能是启动java地址出错 `vi /etc/init.d/jenkins` candidates最后一行地址改为javajdk安装目录 例`/usr/soft/java/jdk1.8.0_161/bin/java`
- tomcat下使用jenkins 将jenkins.war包放到tomcat/webapps下
- 打开ip地址:8080/jenkins
- 根据页面上的提示查看管理员密码 例`cat /root/.jenkins/secrets/initialAdminPassword`
- 初始化插件慢 `find / -name 'default.json sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' /var/lib/jenkins/updates/default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' /var/lib/jenkins/updates/default.json`
- 如果安装jenkins速度很慢,可以使用[清华大学rpm镜像源安装](https://mirrors.tuna.tsinghua.edu.cn/jenkins/redhat-stable/)
- jenkins 相关配置
  - [创建新的任务可看此博客](https://www.cnblogs.com/vipzhou/p/7890016.html)
  - git parameter 分支构建
    - 安装git parameter插件
    - ![image](https://dxy-cos-1258924606.cos.ap-beijing.myqcloud.com/self-img/1596873079.jpg)
    - ![image](https://dxy-cos-1258924606.cos.ap-beijing.myqcloud.com/self-img/2.jpg)


## nginx
- 安装依赖 `yum -y install gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel open openssl-devel`
- 安装nginx `yum -y install nginx`
- nginx操作
  - 设置nginx开机自启 `systemctl enable nginx`
  - 启动 `systemctl start nginx`
  - 查看nginx配置文件地址 `locate nginx.conf`
  - nginx配置
    - 开启反向代理
    ```
        location /api/ {
            proxy_pass http://127.0.0.1:3000/; // 访问服务器ip/api/ 转发到http://127.0.0.1:3000
            proxy_set_header Host $host:$server_port; // 访问服务器ip/api/ 地址改为 服务器ip+端口号
    	}
    ```
    - 配置https(默认配置文件里有对443端口注释掉的,可以去掉注释,然后把申请到的https证书上传到服务器下)
    ```
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
    	server_name  域名地址;
        ssl_certificate crt证书路径;  # 指定证书的位置，绝对路径
        ssl_certificate_key key文件路径;  # 绝对路径，同上

    ```
    - http https兼容(在server80端口下添加)
    ```
        rewrite      ^(.*)$ https://$host$1 permanent;
    ```
    - 其他的一些配置和详细介绍,可自行查询.此处只展示了暂时用到的一些功能.
    - 修改配置文件后重启nginx `nginx -s reload`
    - tips:
      - Job for nginx.service failed because the control process exited with error code(配置文件有可能语句没有以;结尾)
      - Failed to start The nginx HTTP and reverse proxy server(nginx启动时80端口被占用,可以杀死80端口进程,重新启动)
      - nginx 权限问题可执行 `403 chmod -R 777 /root`
