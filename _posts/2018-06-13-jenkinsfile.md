---
layout: post
title:  "jenkinsfile"
date:   2018-06-13
categories: jenkins
---

## 0x01 demo

```groovy
pipeline {
    agent any
    tools {
        maven 'maven3.5.3'
    }
    stages {
        stage ('checkout') {
            steps {
                checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[cancelProcessOnExternalsFail: true, credentialsId: '54f288d2-ee65-4666-9b34-f7f828a50c2f', depthOption: 'infinity', ignoreExternalsOption: false, local: '.', remote: 'https://xxx.xxx.xxx.xxx/xxx']], quietOperation: true, workspaceUpdater: [$class: 'UpdateUpdater']])
            }
        }
        stage ('initialize') {
            steps {
                sh '''
                    echo "PATH = ${PATH}"
                    echo "M2_HOME = ${M2_HOME}"
                '''
            }
        }
        stage ('clean') {
            steps {
                sh 'mvn clean' 
            }
        }
        stage ('deploy') {
            steps {
                sh 'mvn deploy --settings settings.xml' 
            }
        }
        stage ('start') {
            steps {
                sshagent(['kof-trunk']) {
                    sh 'ssh -o StrictHostKeyChecking=no -p 55522 -l root xxx.xxx.xxx.xxx sh start.sh'
                }
            }
        }
    }
}
```
可以看到`jenkinsfile`就是一个groovy的dsl，看起来很直观，`pipeline`里包含多个`step`，每个步骤中可以去触发一些命令或逻辑。上面的demo描述了先使用maven3.5.3这个tool，然后从svn checkout下来代码，接下来去初始化环境变量，然后执行mvn clean、mvn deploy，将打好的包安装到nexus中，最后ssh登录到服务器上去执行更新启动的操作。

## 0x02 坑点
`sshagent`只能使用private key的credentials，这样的话，要先使用`ssh-keygen`生成`private key`和`public key`，然后将public key使用`ssh-copy-id`拷贝到远程服务器上，最后将private key配置到jenkins的credentials中，这样在sshagent中就可以使用这个credentialsId了。

## 0x03 使用方法
[Jenkins2.0 Pipeline使用指南](https://www.nowcoder.com/discuss/69512)