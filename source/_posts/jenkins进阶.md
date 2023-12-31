---
title: jenkins进阶
tags:
  - jenkins
  - 架构师进阶之路
abbrlink: 77c57190
date: 2023-12-31 20:51:55
---

# jenkins进阶
记录最近踩的jenkins大坑，知识点比较零散
## 悲剧的起因
最近计划通过测试覆盖率工具来量化测试工作，避免漏测，少测试，因此选型jenkins+jacoco来实现

## 悲剧时间线
- jenkins插件市场,找到jacoco插件
- 下载jacoco插件，提示当前jenkins版本过低
- 升级jenkins版本
- 重启jenkins失败
- 登陆jenkins机器看日志，提示jdk版本过低
- 安装jdk11,并设置java环境变量
- jenkins 重启成功
- jenkins tool 新增jdk8 和jdk11
- 配置jenkins job，使用 jdk8 
- 运行job，提示编译失败，找不到jdk8的 rt.jar （jdk11没有改jar） 
- job新增打印环境变量，输出结果是 jdk8，但是maven 插件依然使用jdk 11 
- 修改jenkins所在机器的java环境变量为8，重启 jenkins 使用jdk11绝对路径启动 
- 运行job，依然提示失败，使用当前jenkins进程对应的jdk11 
- 回滚jenkins 版本到之前版本（jdk8） 
- 重启jenkins失败，提示config.xml解析异常

## config.xml解析异常
百度和谷歌了一把，都没找到有效的,这里吐槽一下百度，大量重复的内容  
参考了一番搜索结果，查了下jenkins官网资料，最终有效的办法是注释掉报错的xml节点
> xml文件路径在 配置的{jenkins_home}目录下
> 根本原因是 新旧版本xml格式不兼容
> jenkins 所有配置都使用xml方式存储的，遇到坏掉的插件，可以直接注释，或者修改成对应版本的格式
> job配置的xml在{jenkins_home}/jobs/{job_name}/config.xml 
> 只要xml没被删除，可以通过xml还原之前的配置

## maven插件不使用设置的jdk
这个问题如果使用maven项目则比较麻烦(jenkins哪些版本没这个bug,具体不详)，改成使用pipeline构建job
> 理论上来说，编译使用的jdk版本和jenkins进程使用的jdk版本完全是隔离，例如 jenkins 使用11，项目使用8，反之亦然
> 推断原因应该是jenkins的maven项目 使用 scriptEngine 执行shell的时候,写法有问题。

## pipeline 进阶知识点
基础的 jenkins pipeline我不介绍了，搜索一下，或者自己看官网文档。下面列下进阶技巧
### 账户密码托管
jenkins job 有时候会涉及访问远程资源就需要使用对应的账户，例如git账户，oss账户，或者服务器账户  
安全上来讲要统一使用凭证管理
#### 新增凭证
{% asset_img img.png  image %}

#### pipeline 新增凭证参数，并且使用
```groovy
pipeline{
    agent any
    parameters{
        credentials(
                credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl',
                defaultValue: 'admin-user',
                description: ''' 服务器登陆账户 ''',
                name: 'server_user',
                required: true
        )
    }
    stages {
        stage('打印变量') {
            steps {
                withCredentials([usernamePassword(credentialsId:
                        params.server_user, passwordVariable: 'upwd', usernameVariable: 'uname')]) {
                    // 在字符串里面使用${xx}
                    // 在函数里面直接使用变量
                    println(" uname : ${uname} " )
                    println(" upwd : ${upwd} ")
                }
                
            }
        }
       
    }
    
}


```
**打印出来的用户名/密码是星号**

### Publish Over SSH 替代方案
大部分时候jenkins所在的服务器和java服务运行的服务器不是同一个，这个时候就需要上传jar到远程服务，且运行启动脚本  
网上查了下资料，大部分使用publish over ssh,但这个插件已经不维护了，最新版本jenkins插件市场都找不到  
推荐的替代方案是使用 SSH Pipeline Steps 替代
```groovy

pipeline{
    agent any
    parameters{
        credentials(
                credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl',
                defaultValue: 'admin-user',
                description: ''' 服务器登陆账户 ''',
                name: 'server_user',
                required: true
        )
    }
    stages {
        stage('执行远程shell') {
            steps {
                withCredentials([usernamePassword(credentialsId:
                        params.server_user, passwordVariable: 'upwd', usernameVariable: 'uname')]) {
                    def remote = [:]
                    remote.password = upwd
                    remote.user = uname
                    remote.host = "127.0.0.2"
                    remote.name = "127.0.0.2"
                    remote.allowAnyHosts = true
                    writeFile file: 'abc.sh', text: 'ls'
                    // 执行远程命令
                    sshCommand remote: remote, command: 'for i in {1..5}; do echo -n \"Loop \$i \"; date ; sleep 1; done'
                    // 复制当前workspace下文件到 远程机器
                    sshPut remote: remote, from: 'abc.sh', into: '.'
                    // 复制当前 远程机器文件，到当前workspace
                    sshGet remote: remote, from: 'abc.sh', into: 'bac.sh', override: true
                    //  执行远程脚本
                    sshScript remote: remote, script: 'abc.sh'
                    // 删除远程文件
                    sshRemove remote: remote, path: 'abc.sh'
                    
                }
                
            }
        }
       
    }
    
}


```
**当前workspace 在{jenkins_home}/workspace/{job_name}**

### 动态插件方案
有时候有些疑难杂症，难以解决,需要在脚本运行期间使用 jenkins进程相关信息
注意，不要勾选沙箱运行，因为要获取jenkins实例，要突破沙箱限制
```groovy

import  jenkins.model.*
// 有使用的插件也可以import其他的


pipeline{
    agent any
    parameters{
        credentials(
                credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl',
                defaultValue: 'admin-user',
                description: ''' 服务器登陆账户 ''',
                name: 'server_user',
                required: true
        )
    }
    stages {
        stage('使用') {
            steps {

                script{
                    // 获取当前jenkins实例
                    def jks= Jenkins.getInstanceOrNull()
                    println(jks)
                    // 获取指定插件的配置,其他api，查看 https://javadoc.jenkins.io/jenkins/model/Jenkins.html
                    // 理论上所有插件和参数都能查看，也可以动态修改插件
                    def desc=jks.getDescriptor(com.cloudbees.plugins.credentials.Credentials.class)
                    println(desc)
                    def plugin=jks.getPlugin(com.cloudbees.plugins.credentials.Credentials.class)
                    println(plugin)
                }
               
                
            }
        }
       
    }
    
}


```
**高版本的jenkins除了关闭沙箱之外，还需要授权同意**
{% asset_img img-1.png  image %}

## jenkins发布流水线参考demo
```groovy

import jenkins.model.*

/**
 *  see https://javadoc.jenkins.io/
 *  see https://github.com/jenkinsci/ssh-steps-plugin?tab=readme-ov-file#sshput
 */

pipeline {
    agent any

    tools {
        // 多jdk情况下手动指定 jdk，也可以在steps拿到jenkins实例后里面动态切换jdk
        jdk 'jdk8'
    }

    options {
        timestamps()    //设置在项目打印日志时带上对应时间
        disableConcurrentBuilds()   //不允许同时执行流水线，被用来防止同时访问共享资源等
        buildDiscarder(logRotator(numToKeepStr: "5"))   // 表示保留n次构建历史
    }
    parameters {
        // maven home--字符串参数 -- 去掉空白字符串
        string(
                defaultValue: '/usr/local/maven/apache-maven-3.8.1',
                description: ''' maven 目录 ''',
                name: 'maven_home',
                trim: true
        )
        // 发布的机器ip列表 逗号分割
        string(
                defaultValue: '',
                description: ''' 发布的机器ip列表 ''',
                name: 'remote_ip_list_str',
                trim: true
        )
        // git_branches git 分支
        string(
                defaultValue: '*/test',
                description: ''' git 分支 ''',
                name: 'git_branches',
                trim: true
        )
        // git_time_out 单位分钟
        string(
                defaultValue: '20',
                description: ''' git_time_out ''',
                name: 'git_time_out',
                trim: true
        )
        // git_url
        string(
                defaultValue: '',
                description: ''' git 仓库地址 ''',
                name: 'git_url',
                trim: true
        )
        // ssh 凭证，访问git使用
        credentials(
                credentialType: 'com.cloudbees.jenkins.plugins.sshcredentials.impl.BasicSSHUserPrivateKey',
                defaultValue: 'gitlab_jenkins',
                description: ''' ssh 凭证 ''',
                name: 'credentials_id',
                required: true
        )
        booleanParam(
                defaultValue: false,
                description: '是否启用测试覆盖率',
                name: 'is_enable_jacoco'
        )
        credentials(
                credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl',
                defaultValue: 'server-user-admin',
                description: ''' 服务器登陆账户 ''',
                name: 'server_user',
                required: true
        )


    }

    stages {
        stage('打印环境变量') {
            steps {
                println("=========开始打印环境变量=========")
                sh "printenv"
            }
        }

        stage('拉取git代码') {
            steps {
                script {
                    def branches = params.git_branches
                    def gitTimeOut = params.git_time_out
                    def credentialsId = params.credentials_id
                    def gitUrl = params.git_url
                    println("========开始拉取git代码=======")
                    println(gitUrl)
                    checkout scmGit(branches: [[name: branches]],
                            extensions: [cloneOption(noTags: false, reference: '', shallow: true, timeout: gitTimeOut)], userRemoteConfigs: [[credentialsId: credentialsId, url: gitUrl]])

                }

            }
        }
        stage('maven编译') {
            steps {

                script {
                    // /usr/local/maven/apache-maven-3.8.1
                    println("========开始maven编译=======")
                    sh """
                    pwd
                    ${params.maven_home}/bin/mvn clean install -T 1C -Dmaven.test.skip=true -Dmaven.compile.fork=true -U -B -f ${env.WORKSPACE}/pom.xml
                    """
                }

            }
        }


        stage('部署代码') {
            steps {
                script {
                    // 获取当前Jekins 实例，理论上非空
//                    def inst = Jenkins.getInstanceOrNull()

                    withCredentials([usernamePassword(credentialsId: params.server_user, passwordVariable: 'upwd', usernameVariable: 'uname')]) {
                        def remote = [:]
                        def serveripList = params.remote_ip_list_str.split(",")
                        for (ip in serveripList) {
                            remote.password = upwd
                            remote.user = uname
                            remote.host = ip
                            remote.name = ip
                            remote.allowAnyHosts = true

                            println(remote)
                            sshPut remote: remote, from: 'test.jar', into: '/home/target/jenkins'
                            sshScript remote: remote, script: '/home/back/java_start.sh'

                            while (true) {
                                println("=== 开始健康检查===")
                                try {
                                    def response = sh(
                                            script: """  curl -X GET "http://${ip}:8191/test/health/check" """,
                                            returnStdout: true
                                    ).trim()
                                    if (response.contains("SUCCESS")) {
                                        break
                                    }
                                    println(" 健康检查失败 sleep 10 s")
                                    sleep(10)
                                } catch (exec) {
                                    println(" 健康检查失败  ${exec}  sleep 10 s ")
                                    sleep(10)
                                }
                            }
                            println("====== 健康检查通过 =====")

                        }
                    }

                }

            }

        }

    }
}


```







