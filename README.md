# Koa-Typescript-Mongodb
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
* [linux环境搭建](#linux服务器搭建)
  * 安装javajdk
  * 安装node
  * 安装mongodb
* [jenkins - 持续集成,自动化部署](#jenkins)
* [nginx相关操作](#nginx)
* [git相关操作](#git相关操作)
  * 把本地文件夹设置成git远程仓库
* [对象存储 - 文件上传](#对象存储)
  * 阿里云oss
  * 腾讯云cos
* [HTTP缓存](#http缓存)
  * 强缓存
  * 协议缓存
  * 优先级
  * 不能缓存的请求
* [日志持久化到数据库](#日志持久化到数据库)
  
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

## linux服务器搭建
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

## jenkins
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


## git相关操作
- 把本地文件夹设置成git远程仓库
    1. `git init`
    2. `git remote add origin git 地址`
    3. `git remote set-url origin git https地址`
    4. `git pull origin master --allow-unrelated-histories`合并两个独立启动仓库的历史


## 对象存储
- 阿里云oss
    1. 创建 bucket
    2. 获取accesskeyid和accesssecret
    3. nodejs使用ali-oss
    ```
        npm i ali-oss
        
        src/controllers/admin/file.ts
        
        import OSS from 'ali-oss'
        let client = new OSS({
            accessKeyId: accessKeyId,
            accessKeySecret: accessKeySecret,
            bucket: 创建的bucketname,
            region: oss所属地区,
            cname: true, // 是否使用自定义域名
            endpoint: 'dxy-oss1.oss-cn-beijing.aliyuncs.com'
        })
        
        const file = ctx.request.files.file // 接收传输到后端的blob文件流
        let result = await client.put(file.name, file.path);
    ```
- 腾讯云cos
    1. 创建存储桶
    2. 获取SecretId和SecretKey
    3. nodejs使用腾讯云cos
    ```
        npm i cos-nodejs-sdk-v5
        
        src/controllers/admin/file.ts
        
        const COS = require('cos-nodejs-sdk-v5')
        const cos = new COS({
            SecretId: SecretId,
            SecretKey: 获取SecretId和SecretKey
        });
        
        const file = ctx.request.files.file
        const params = {
            Bucket: 存储桶名称,
            Region: cos所属地区,
            Key: file.name, // 随意确认的文件key
            FilePath: file.path // 文件路径
        }
        
        cos.sliceUploadFile(params, (err: any, data: any) => {
            console.log(err, data)
        })
        
    ```


## http缓存

- 强缓存
    - Expires: 服务端返回的到期时间,在响应http请求时告诉浏览器在过期时间前浏览器可以直接从浏览器缓存取数据,而无需再次请求.
        - 缺点:当客户端时间和服务端时间不一致时,到期时间根据服务端时间.
    - Cache-Control: 定义所有缓存机制都必须遵循的缓存指示,包括public、private、no-cache(表示可以存储，但在重新验证其有效性之前不能用于响应客户端请求)、no-store、max-age、s-maxage以及must-revalidate
    - 优先级： Cache-Control > Expires
    - 用法: 
    ```
        ctx.response.set('cache-control', `public, max-age=${60 * 10}`)
        ctx.response.set('expires', new Date(Date.now() + 2 * 60 * 1000).toString());
    ```
    - 更新强缓存: 
        - 缓存到期
        - 服务器上的文件名称更改(打包后文件名加hash值)
        
- 协议缓存
    - Last-Modified/If-Modified-Since
        - Last-Modified: 服务器在响应请求时，告诉浏览器资源的最后修改时间。
        - If-Modified-Since: 再次请求服务器时，通过此字段通知服务器上次请求时，服务器返回的资源最后修改时间。服务器收到请求后发现有头If-Modified-Since则与被请求资源的最后修改时间进行比对。若资源的最后修改时间大于If-Modified-Since，说明资源又被改动过，则响应整片资源内容，返回状态码200；若资源的最后修改时间小于或等于If-Modified-Since，说明资源无新修改，则响应HTTP304，告知浏览器继续使用所保存的cache。
        - 缺点: Last-Modified 标注的最后修改时间只能精确到秒，如果有些资源在一秒之内被多次修改的话，他就不能准确标注文件的新鲜度了如果某些资源会被定期生成，当内容没有变化，但Last-Modified却改变了，导致文件没使用缓存有可能存在服务器没有准确获取资源修改时间，或者与代理服务器时间不一致的情形
    - Etag/If-None-Match
        - Etag: 服务器资源的唯一标识符, 浏览器可以根据ETag值缓存数据, 节省带宽
        - If-None-Match:再次请求服务器时，通过此字段通知服务器客户段缓存数据的唯一标识.服务器收到请求后发现有头If-None-Match则与被请求资源的唯一标识进行比对.
    - 优先级: Etag/If-None-Match > Last-Modified/If-Modified-Since
    - 用法: 
    ```
        const ifNoneMatch = ctx.header['if-none-match']
        const ifModifiedSince = ctx.header['if-modified-since']
        if (ifNoneMatch && ifModifiedSince && !checkEtag(ctx, ifNoneMatch, doc)) {
            ctx.status = 304
        } else {
            ctx.body = {
                errCode: 0,
                data: {
                    list: doc
                },
                message: 'success'
            }
    
            ctx.response.set('cache-control', `no-cache`)
            ctx.response.set('Content-Type', `application/json`)
            ctx.response.set('Last-Modified', `${new Date()}`)
            ctx.response.set('Etag', `${crypto.createHash('md5').update(doc.toString()).digest('hex')}`)
        }
        
        checkEtag(ctx, ifNoneMatch, doc) {
            if (ifNoneMatch) {
                // 要去检查文件是否更改，生成md5（hash），再转化为 hex （16进制）
                const etag = crypto.createHash('md5').update(doc.toString()).digest('hex')
                // 没更改
                if (ifNoneMatch === etag) {
                    // 304 继续使用缓存
                    return false
                } else {
                    return true
                }
            } else {
                return true
            }
        }
    ```
    
- 优先级: 强缓存 > 协议缓存
- 不能缓存的请求
    1. 不能被缓存的请求HTTP信息头中包含Cache-Control:no-cache，pragma:no-cache，或Cache-Control:max-age=0 等告诉浏览器不用缓存的请求
    2. 需要根据Cookie，认证信息等决定输入内容的动态请求是不能被缓存的
    3. HTTP 响应头中不包含 Last-Modified/Etag，也不包含 Cache-Control/Expires 的请求无法被缓存
    4. 目前浏览器的实现是不会对POST请求的响应做缓存的（从语义上来说也不应该），并且规范中也规定了返回状态码不允许是304.如果在POST请求对应的响应中包含Freshness相关信息的话，这次响应也是可以被缓存.


## 日志持久化到数据库
- `npm i log4js`
- 正常请求
```
    src/util/log.ts
    
    import log4js from 'log4js'
    let resLogger = log4js.getLogger('response')
    
    const log2db = (msg: String, level: String = 'info', info: any) => {
        let log = {
            level: level,
            message: msg,
            info: {
                method: info.method,
                url: info.url,
                costTime: info.costTime,
                body: JSON.stringify(info.body),
                status: level === 'info' ? info.response.status : info.err.status,
                response: {
                    status: info.response.status,
                    message: level === 'info' ? info.response.message : info.err.message,
                    header: JSON.stringify(info.response.header),
                    body: JSON.stringify(info.response.body)
                }
            }
        }
        Log.create(log, (err: any, res: any) => {
            if(err) {console.log(err)}
        })
    }
    
    const formatError = (ctx: any, err: any, costTime: any) => {
        const {method, url, body, response, status} = ctx
        return {method, url, body, costTime, err, response, status}
    }
    
    const formatRes = (ctx: any, costTime: any) => {
        const {method, url, body, response} = ctx
        return {method, url, body, costTime, response}
    }
    
    // 封装响应日志
    resLogger = (ctx: any, resTime: any) => {
        if(ctx) {
            log2db('RequestInfo', 'info', formatRes(ctx, resTime))
            resLogger.info(formatRes(ctx, resTime))
        }
    }
    
    src/server.ts
    app.use(async (ctx: { method: any; url: any }, next: () => any) => {
        const start = Number(new Date())
        await next()
        const end = Number(new Date())
        const ms = start - end
        console.log(`${ctx.method} ${ctx.url} - ${ms}ms`)
      + log4js.resLogger(ctx, ms)
    })
```
- 错误日志
```
    src/util/log.ts
    let errorLogger = log4js.getLogger('error')
    // 封装错误日志
    errLogger = (ctx: any, error: any, resTime?: any) => {
        if(ctx && error) {
            log2db('ErrorRequest', 'error', formatError(ctx, error, resTime))
            errorLogger.error(formatError(ctx, error, resTime))
        }
    }
    
    src/server.ts
    app.on('error', async (err, ctx) => {
      + log4js.errLogger(ctx, err)
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
- 日志Schema
```
    import mongoose from 'mongoose'
    import moment from 'moment'
    
    const { Schema, model } = mongoose
    
    const logSchema = new Schema({
        level: {
            type: String
        },
        message: {
            type: String
        },
        info: {
            method: String,
            url: String,
            costTime: Number,
            body: String,
            status: Number,
            response: {
                status: Number,
                message: String,
                header: String,
                body: String
            }
        },
        createDate: {
            type: Date,
            default: Date.now(),
            get: (v: any) => moment(v).format('YYYY-MM-DD HH:mm:ss')
        }
    })
    
    logSchema.set('toJSON', {getters: true})
    
    export default model('log', logSchema, 'logs')
```
