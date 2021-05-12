#### 一 环境

- [ ] windows10
- [ ] nexus-3.20.1-01-win64
- [ ] node-12.18.3
- [ ] npm-6.14.6



#### 二 安装

###### 2.1 在官网下载[Nexus Repository Manager OSS 3.x](https://www.sonatype.com/download-oss-sonatype), 解压至任意位置

###### 2.2 切换到 nexus-3.20.0-01/bin 目录并执行

```shell
./nexus.exe /run 运行服务, 第一次要初始化，需要耐心等待
```

###### 2.3 启动完毕后，访问http://localhost:8081/，点击右上角 `Sign In` 登陆, 默认账号: admin 密码: admin123，



#### 三 添加npm仓库

| 仓库名称      | 类型        | 作用                                          |
| ------------- | ----------- | --------------------------------------------- |
| npm-proxy     | npm(proxy)  | 对公网镜像的代理，比如taobao镜像或者npmjs镜像 |
| npm-releases  | npm(hosted) | 存放发布包，不可覆盖发布                      |
| npm-snapshots | npm(hosted) | 存放快照包，可覆盖发布                        |
| npm-public    | npm(group)  | 仓库组，对外                                  |
|               |             |                                               |

#### 四 从仓库下载包

1. 配置npm仓库，npm config set registry http://localhost:8081/repository/npm-public/

2. 初始化项目，查看是否从自己的仓库地址拉取包

   ```
   mkdir npm-demo && cd npm-demo
   npm init -y
   npm --loglevel info install grunt
   ```

3. 设置权限, Realms 菜单, 将 npm Bearer Token Realm 添加到右边

4. 用户登录

   ```shell
   ## 添加新用户登陆
   npm adduser -registry http://127.0.0.1:8081/repository/npm-public/
   ## 直接使用已有用户登陆（推荐）
   npm login --registry=http://127.0.0.1:8081/repository/npm-public/
   ```

   

#### 五 发布包到仓库

确保要发布的模块跟目录有 package.json 文件

1. 直接发布

   ```json
   #### 发布命令
   npm publish –registry http://127.0.0.1:8081/repository/npm-snapshots/
   ```

2. 通过配置package.json发布

   ```json
   #### package.json
   {
   	"publishConfig": { "registry": "http://127.0.0.1:8081/repository/npm-snapshots/" }
   }
   
   #### 发布命令
   npm publish
   ```

   

#### 六 参考

- https://www.sonatype.com/products/repository-oss-download
- https://www.cnblogs.com/xueyoucd/p/9538126.html

