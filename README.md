# 公司私有Maven仓库搭建
## 1.为什么要搭建maven私有仓库
平时做android开发，用得最多的是gradle。其实gradle的第三方库，也是放在maven仓库上。对于第三方库，大家基本都配置maven、gradle从远程获取，估计很少直接下载jar放在工程里（对于没有放在maven repository上的库，只能这么干）。这么做方便管理依赖。

但有时候依赖太多，每次编译都得去各个公共仓库确认，导致编译时间变长，有时候网络差的时候甚至一直会卡在那里。这时有一个私有的Maven仓库作为缓存，将会大大减少编译时间，本机请求第三方库时，将会先请求私有仓库，如果私有仓库已经有缓存，将直接返回，如果没有，那么私有仓库会先向共有仓库请求并缓存下来，然后返回给本机。

做app开发，特别是只有几万行代码量的小项目，开发团队也就几个人，通常只用一个工程玩耍。随着业务扩展，工程变得越来越大，代码量大大增加，开发人数也多了，问题开始暴漏：改动一个地方往往影响到其他人的代码，功能模块耦合严重，构建速度慢....

业界一些解决方法：
1. 组件化，按功能拆分出各种组件，数据存储、网络层、日志 等；
2. 拆分业务，一个业务一个module；
3. 业务插件化，一个业务一个工程，每个业务独立编译并运行.....

因此，引入依赖管理是必不可少的。把各个模块单独编译，部署上maven仓库，主工程or业务工程通过maven、gradle引用这些依赖。这么做还有好处，就是持续集成！某个模块修改了，跑单元测试，通过后才放上仓库。业务工程同步一下maven，万一有问题，还可以在服务端回滚到上一个版本。

所以我们希望通过搭建一个私有maven仓库，来提高我们的开发效率。

## 2. 使用Nexus搭建maven私服
**Nexus是什么？**
Nexus是一个基于maven的仓库管理的社区项目。主要的使用场景就是可以在局域网搭建一个maven私服，用来部署第三方公共构件或者作为远程仓库在该局域网的一个代理。简单举几个例子就是：
第三方Jar包可以放在nexus上，项目可以直接通过Url和路径配置直接引用，方便进行统一管理
同时有多个项目在开发的时候，一些共用基础模块可以单独抽取到nexus上，需要用的项目直接从nexus上拉取就行(基础模块的实现，维护和部署可以交给专门的人员，其他项目不用关心代码实现，这样也可以达到保证核心代码不泄露)
封闭开发的过程中开发机是不能上公网的，所以连接central repository和下载jar就比较麻烦，这时就可以用nexus搭建起来一个介于公网和局域网之间的桥梁

**Nexus下载**
官网下载地址：https://www.sonatype.com/download-oss-sonatype
这里运行环境是Ubuntu，选Unix下载
软件运行所需环境：JDK

**Nexus安装**
下载nexus-3.13.0-01-unix.tar.gz之后，采用FileZilla到服务器上（或使用scp命令），在/usr/local/目录下新建nexus目录，解压安装，后台运行，然后浏览器访问http://服务器地址:8081（这里是http://10.200.10.224:8081/）

**Nexus配置**
进入主页如下图，点击右上角登录，默认用户admin，密码admin123。
![](http://ww1.sinaimg.cn/large/8c306a57gy1fvg2ppif7lj21h60pr4qp.jpg)

这里新建一个用户pptv，如下图
![](http://ww1.sinaimg.cn/large/8c306a57gy1fvg2r2rx9uj20wk098jxe.jpg)

下面简单介绍下Nexus的界面
**Blob Stores**
文件存储的地方，创建一个目录的话，对应文件系统的一个目录，如图所示：
![](http://ww1.sinaimg.cn/large/8c306a57gy1fvg315wwkpj20hk07zmzk.jpg)

可对不同的repository设置不同的blog，相当于不同的库存在不同的文件目录下，如缓存的第三方库存在default，本地上传的库存在如图所示path目录。
此功能不设置也没关系

**Repositories**
如图所示是一个新建用户的默认Repositories
![](http://ww1.sinaimg.cn/large/8c306a57gy1fvg34qfj6vj20qs09e438.jpg)
1. Proxy
这里就是代理的意思，代理中央 Maven 仓库，当 PC 访问中央库的时候，先通过 Proxy 下载到 Nexus 仓库，然后再从 Nexus 仓库下载到 PC 本地。
这样的优势只要其中一个人从中央库下来了，以后大家都是从 Nexus 私服上进行下来，私服一般部署在内网，这样大大节约的宽带。

2. Hosted
托管仓库，本地上传的库放这里
Hosted 有三种方式，Releases、SNAPSHOT、Mixed
Releases: 一般是已经发布的Jar 包
Snapshot: 未发布的版本，快照
Mixed：混合的

3. Group
把上面两种仓库结合起来，原理如下图
![](http://ww1.sinaimg.cn/large/8c306a57gy1fvg3a45kplj20qe04sjur.jpg)

## Android Studio使用配置方法
### 1.基本使用-缓存功能
在project的build.gradle文件中增加maven仓库组地址即可
```
allprojects {
    repositories {
        //maven仓库组地址
        maven{
            url "http://10.200.10.224:8081/repository/maven-public/"
        }
        google()
        jcenter();
    }
}
```

### 2.上传本地库
首先，在gradle.properties配置如下：
```
#Maven仓库地址
MAVEN_REPO_RELEASE_URL = http://10.200.10.224:8081/repository/maven-releases/
MAVEN_REPO_SNAPSHOT_URL = http://10.200.10.224:8081/repository/maven-snapshots/

#登录nexus的用户名
NEXUS_USERNAME=pptv
#登录nexus的密码
NEXUS_PASSWORD=pptv123
#groupid（最终你引用时的名字）
GROUP_ID=pptv
#type
TYPE=aar
#description
DESCRIPTION=maven-lib
```
然后，修改module对应的build.gradle文件，添加以下配置
```
uploadArchives {
    configuration = configurations.archives
    repositories {
        mavenDeployer {
            snapshotRepository(url: MAVEN_REPO_SNAPSHOT_URL) {
                authentication(userName: NEXUS_USERNAME, password: NEXUS_PASSWORD)
            }
            repository(url: MAVEN_REPO_RELEASE_URL) {
                authentication(userName: NEXUS_USERNAME, password: NEXUS_PASSWORD)
            }
            pom.project {
                //版本，有更新时修改版本号，再上传
                //发布快照版本，在版本号后加-SNAPSHOT，必须是大写
//                version '1.0.1'
                version '1.0.1-SNAPSHOT'
                //名字
                artifactId 'maven-lib'
                groupId GROUP_ID
                packaging TYPE
                description DESCRIPTION
            }
        }
    }
}

artifacts {
    archives file('maven-lib.aar')
}
```
最后，点击uploadArchives上传即可
![](http://ww1.sinaimg.cn/large/8c306a57gy1fvg5ahp4f6j20f90bpwid.jpg)

注意点：
- 第一次上传会有个红色错误，正常，第二次就不会了
- release同一版本号的只能上传一次，上传第二次需要修改版本号，否则上传失败
- SNAPSHOT可以上传多次

### 3.引用
在app的build.gradle文件中像引用第三方库一样引用maven-lib，如下图，这里引用的是release版本，也可以引用SNAPSHOT版本
![](http://ww1.sinaimg.cn/large/8c306a57gy1fvg5i0bgn0j20gr04dwgd.jpg)

查看引用的来源
![](http://ww1.sinaimg.cn/large/8c306a57gy1fvg5lcqgdfj20b203yt9q.jpg)

**如果还有问题，请查看**[https://github.com/kangisme/MavenDemo](https://github.com/kangisme/MavenDemo)