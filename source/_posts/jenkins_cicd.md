---
title: 基于jenkins实现CI/CD自动化
date: 2026-11-24 15:35:41
slug: jenkins-cicd
tags:
  - 技能
categories:
  - 随笔
description: 基于jenkins实现CI/CD自动化实现介绍
toc: true
---
本文介绍了Jenkins + Git自动化部署流程搭建方法。先说明了在CentOS 7环境下，提供war包手动安装和docker镜像安装两种Jenkins安装方式，接着阐述了Jenkins的配置、项目配置，包括选择仓库、配置webhook等，最后提到可选的构建通知配置，如集成企业微信或钉钉机器人。

## 运行逻辑
1、git代码管理，git push推送上传时触发CI/CD自动化流程
2、代码push后自动触发构建：在Git仓库（GitLab/Gitea/GitHub）配置Webhook，指向Jenkins的触发URL，格式如 http://jenkins-ip:8080/job/myapp/build?token=your-token。
3、CI/CD自动化流程 包括：
  代码规范检查 -> 准备运行环境 -> 编译打包 -> 构建部署 -> 运行自动化用例（核心）-> 发布

## 环境部署
1、‌安装Jenkins‌：推荐用Docker容器化部署，方便迁移和升级。启动时需挂载Docker socket（若需构建镜像）和Jenkins数据目录。
2、‌安装必要插件‌：Git、Pipeline、SSH（Publish Over SSH）、Docker Pipeline，以及Locale（中文界面）。
‌3、配置全局工具‌：在“全局工具配置”里设置Git路径，以及C编译工具链（gcc、make）。若用容器，需确保容器内有编译环境。‌‌

## 具体实施
将构建流程写成Jenkinsfile放进代码仓库，流程可版本化、可追溯。‌‌
  a‌、新建任务‌：选择“流水线”类型。
  ![image](/img/jenkins_setting.png  "jenkins设置")
  b‌、配置Git仓库‌：填入仓库地址和凭据（推荐SSH密钥方式，更稳定）。
  ![image](/img/git_setting.png  "git设置")
  ‌c、编写Jenkinsfile‌：核心流程分几个阶段。‌‌

``` bash
Jenkinsfile示例：

pipeline {
    agent any
    
    environment {
        // 部署目标服务器信息
        DEPLOY_HOST = '192.168.1.100'
        DEPLOY_DIR = '/opt/myapp'
        APP_NAME = 'myapp'
    }
    
    stages {
        stage('拉取代码') {
            steps {
                checkout scm
            }
        }
        
        stage('编译构建') {
            steps {
                // C项目用make构建，需确保Jenkins环境装了gcc和make
                sh 'make clean && make'
            }
        }
        
        stage('部署') {
            steps {
                // 方式一：通过SSH推送二进制并重启服务
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'deploy-server',
                            transfers: [
                                sshTransfer(
                                    sourceFiles: 'bin/myapp',
                                    remoteDirectory: DEPLOY_DIR,
                                    execCommand: """
                                        sudo systemctl stop myapp || true
                                        sudo systemctl start myapp
                                    """
                                )
                            ]
                        )
                    ]
                )
                
                // 方式二：构建Docker镜像并部署（若项目容器化）
                // sh 'docker build -t myapp:${BUILD_NUMBER}.'
                // sh 'docker run -d --name myapp -p 8080:80 myapp:${BUILD_NUMBER}'
            }
        }
    }
    
    post {
        success {
            echo "构建部署成功，版本：${BUILD_NUMBER}"
        }
        failure {
            echo "构建失败，请查看控制台日志"
        }
    }
}

```
