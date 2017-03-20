---
layout: post
title:  "为Rails项目搭建Jenkins CI服务器(一)"
date:   2017-03-20 22:30:00 +0800
categories: other
---

## 概述

最近重新搭建了一次Jenkins CI服务，并尝试了使用pipeline配置任务，结合以前Jenkins的配置经验，稍微做了一下总结。本文将讲述如何从头开始搭建一个Jenkins服务器并进行相关配置，接着会介绍配置Jenkins任务的几种方式，最后会列举一个rails + jenkins + bitbucket的CI任务配置实例。

## 搭建jenkins服务器

以下搭建步骤是在ubuntu14.04下进行，其他环境的搭建步骤参考官方的[文档](https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins)。

### 安装jenkins

参照[文档](https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins+on+Ubuntu)，在ubuntu下安装步骤比较简单:

```shell
wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```

### jenkins访问配置

Jenkins默认使用`8080`端口，若要使用其他端口可以在`/etc/default/jenkins`文件下修改这一行:

```
HTTP_PORT=8080
```

为了访问Jenkins服务器，我们要将80端口的请求代理到Jenkins使用的端口，参考官方提供的Nginx配置: 

```
upstream app_server {
	# 若jenkins使用其他端口，127.0.0.1:8080要作对应修改
    server 127.0.0.1:8080 fail_timeout=0;
}

server {
    listen 80;
    listen [::]:80 default ipv6only=on;
    server_name ci.yourcompany.com;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;

        if (!-f $request_filename) {
            proxy_pass http://app_server;
            break;
        }
    }
}
```

假如你需要使用ssl，参考这个[配置](https://gist.github.com/rdegges/913102#gistcomment-198697)

### 注意事项

* Jenkins安装完成后会自动创建一个名为`jenkins`的用户，用户的根目录位于`/var/lib/jenkins`。
* Jenkins在执行任务时会以这个`jenkins`用户执行。

## Jenkins初始化与配置

### 初始化

上一步Jenkins访问配置完成之后，访问Jenkins服务器的地址`ci.yourcompany.com`。首次访问需要进行初始化:

 ![unlock]({{ sites }}/assets/2017-03-20/unlock.png)

根据提示执行以下命令取得初始化密码。

```shell
cat /var/lib/jenkins/secrets/initialAdminPassword
```

输入密码后需要设置Jenkins的管理员账号，并且选择需要安装的插件，插件可以在进入系统之后管理，因此此处按默认配置即可。

至此Jenkins服务器已经可用，接下来介绍一些常用但非必要的配置。

### shell配置

Jenkins在任务运行过程中可以执行shell脚本，但在使用前请先检查默认配置的shell是否是你想要的。进入Manage Jenkins -> System Configuration ，检查`Shell`选项。

### ssh密钥配置

在CI执行的过程中需要使用到的密钥可以在首页的`Credentials`中配置，在对应的Domain下选择`Add credentials`，进入如下页面:

![ssh_cert]({{ sites }}/assets/2017-03-20/ssh_cert.png)

我们需要添加ssh密钥的配置，类型选择`SSH Username with private key`。private key的提供方式有三种，第一种`Enter directly`是直接填到Jenkins的配置中，后两种需要在服务器上生成ssh密钥。例如采用第三种`From the Jenkins master ~/.ssh`，需要像这样生成ssh密钥：

```shell
sudo su jenkins
ssh-keygen -t rsa
```

选择好配置之后点`OK`，以后在编写任务的时候就可以使用该密钥。

### 管理项目配置文件

项目中的密钥配置不会放在代码库中，可以使用Jenkins的插件[Config File Provider Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Config+File+Provider+Plugin)。安装插件后进入Manage Jenkins -> Managed files即可创建配置文件。

## 编写Jenkins任务

编写Jenkins任务主要有两种方式:
1. 使用Jenkins提供的freestyle project
2. 用户编写pipeline

freestyle project上手简单，基本上参考一个例子就能掌握，但需要按照给定的步骤执行CI流程，不够灵活。pipeline使用Groovy编写，需要用户掌握一点Groovy语法，而且插件文档不太完善，某些插件可能没提供在pipeline中使用的文档，但pipeline最大的优势是灵活，用户可以自行控制整个CI的流程。下面将以配置一个Rails项目的CI任务来讲解。

### Freestyle project

新建一个`freestyle project`，进入配置。整个任务可以分为六个部分:

* General
* Source Code Management
* Build Triggers
* Build Enviroments
* Build
* Post-build Actions

首先要说明的是，对于每个任务，Jenkins都会生成一个workspace，并会把项目代码clone到workspace中，在配置中的路径的当前路径均是对应workspace的目录(即项目的根目录)。

#### General

项目的基础信息配置，在每个选项旁都有一个问号，点开后可以看到详细的文档介绍。

#### Source Code Management

选择拉取代码的方式。我们这里选择Git，使用SSH方式拉取，`Repository URL`填写项目的SSH地址，Credentials处需要选择之前添加的SSH密钥(PS: 不要忘了在代码托管服务中添加该SSH密钥的公钥)。配置如图:

 ![source_code_management]({{ sites }}/assets/2017-03-20/source_code_management.png)

#### Build Triggers

该任务的触发方式，不作设置的话只能手动触发，但可以配置为在某个任务结束之后触发，或者是在代码托管收到commit时通知Jenkins，可能需要另外安装插件。此外还能选择采用轮询的方式(不推荐)。

#### Build Enviroments

这个步骤会为build准备环境，此处我们先安装[RVM Plugin](https://wiki.jenkins-ci.org/display/JENKINS/RVM+Plugin)，之后我们就能看到`Run the build in a RVM-managed environment`的选项，作以下配置:

![rvm]({{ sites }}/assets/2017-03-20/rvm.png)

#### Build

该部分配置CI过程执行什么的地方(例如跑测试，生成报告等等)。点击`add build step`添加构建步骤。如果安装了`Config File Provider Plugin`，则可以选择`Provide Configuration files`，选择配置文件后填好路径，这样在任务执行时配置文件就会被复制到目标路径。 配置如图:

![provide_config_file]({{ sites }}/assets/2017-03-20/provide_config_file.png)

接下来需要编写shell脚本，添加`Execute shell`，编写以下shell script:

```shell
source ~/.bashrc
gem install bundler
bundle install

# 重新建立数据库
bundle exec rake db:drop RAILS_ENV=test
bundle exec rake db:create RAILS_ENV=test
bundle exec rake db:migrate RAILS_ENV=test

# 执行测试
bundle exec rake test
```

#### Post-build Actions

Build结束后要执行的动作，一般可以用于向代码托管服务提交构建结果，发送email，发布HTML页面等等，有一部分需要插件。这个例子就不配置了。

以上就是Freestyle project的简单配置流程，基本可以满足接收通知->构建->测试->结果反馈的流程，而且也有各种插件支持。

### Pipeline

新建一个`Pipeline`项目，可以看到只有四个部分:

* General
* Build Triggers
* Advanced Project Options
* Pipeline

General和Build Triggers跟freestyle project中的是一样的，Advanced Project Options默认只有一个设置显示名。基本上全部的配置都集中在Pipeline。pipeline的脚本可以直接写在Jenkins任务的配置中，也可以放在项目根目录`Jenkinsfile`文件里。最佳实践是在项目中创建`Jenkinsfile`并加入版本控制中。

#### Pipeline script

用户可以使用两种方式编写[Pipeline script](https://jenkins.io/doc/book/pipeline/#overview)：声名式(declarative)和脚本式(scripted)。声明式使用DSL来描述pipeline，而脚本式则完全是使用Groovy来编写。个人认为声明式并没有怎样简化pipeline的编写，只是提高了可读性。总体看来脚本式和声明式差不多，下面编写流程将采用脚本式pipeline，对声明式感兴趣的话可以浏览[官方文档](https://jenkins.io/doc/book/pipeline/syntax/)。

对于脚本式pipeline，结构大体上是这样的：

```groovy
node {
	stage('Build') {
		sh 'make'
	}
	stage('Test') {
		sh 'make check'
	}
	stage('Deploy') {
		sh 'make publish'
	}
}
```

用户可以定义各种阶段(stage)进行不同的工作。这里先介绍一下怎样快速上手pipeline。在项目的页面左侧能找到`Pipeline Syntax`的选项，可以进入一个生成pipeline script的页面，相当于查看文档，当安装了插件后，部分插件的方法也会被添加到这里。用的比较多的是Jenkins提供的`sh`方法，可以执行给定的shell脚本，并且可以设置把stdout作为返回值，具体参考[文档](https://jenkins.io/doc/pipeline/steps/workflow-durable-task-step/#code-sh-code-shell-script)。pipeline生成页面如图：

 ![pipeline_generator]({{ sites }}/assets/2017-03-20/pipeline_generator.png)

接下来编写一个跟上面的freestyle project相同的pipeline任务。首先必须确保CI运行环境，要先安装rvm。可以在pipeline中进行安装，我选择的是在服务器上直接装好：

```shell
sudo su jenkins
cd
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
\curl -sSL https://get.rvm.io | bash
```

pipeline的结构如下：

```groovy
node{
	def rails_env = 'test'
	stage('Preparation'){
      //...
	}
	stage('Test'){
      //...
	}
}
```

先看Preparation阶段。首先要做的工作是从代码库中clone代码，从generator生成如下的方法调用：

```groovy
git branch: 'master', credentialsId: 'b33c1c57-737d-4195-a6f5-446a9d000f2a', url: 'git@jenkins/example.git'
```

接下来是准备测试环境，为了让pipeline更加灵活，可以通过项目里的.ruby-version文件安装和使用指定版本的ruby。假如想要使用特定版本的bundler进行`bundle install`，可以通过检测`Gemfile.lock`中的信息来获取bundler的版本(适用于大于1.10版本的bundler生成的`Gemfile.lock`)。上述步骤完成后需要先删除上次任务执行时的数据库并重新创建。编写如下pipeline:

```groovy
node{
	stage('Preparation'){
		// 拉取代码
		git branch: 'master', credentialsId: 'b33c1c57-737d-4195-a6f5-446a9d000f2a', url: 'git@jenkins/example.git'
		
		// 获取项目ruby版本
		def rubyVersion = sh(script: 'cat .ruby-version', returnStdout: true).trim()

		sh 'source $HOME/.bashrc'

		// 假如没有安装所需版本ruby，用rvm进行安装
		// "${foo}"为groovy的字符串插值方式
		// """也是groovy中表示字符串的方式
		sh """if ! rvm ${rubyVersion} do ruby -v &> /dev/null; then
		rvm get stable
		rvm install ${rubyVersion}
		fi"""

		// 获取bundler版本并进行安装
		def bundlerVersion = sh(script: '''rvm all do ruby -e 'puts $<.read[/BUNDLED WITH\\n   (\\S+)$/, 1] || "<1.10"' Gemfile.lock''', returnStdout: true).trim()
		sh "rvm ${rubyVersion} do gem install bundler --conservative --no-document -v ${bundlerVersion}"
		// 在CI环境下适合使用--deployment选项，具体看:http://bundler.io/v1.14/man/bundle-install.1.html#DEPLOYMENT-MODE
		sh "rvm ${rubyVersion} do bundle install --deployment --retry=3"

		// 使用Config File Provider Plugin提供的方法复制配置文件
		configFileProvider([configFile(fileId: 'cd003fbc-d1ae-496b-b68c-b08f0640a286', targetLocation: 'config/secrets.yml', variable: 'SECRET_FILE'), configFile(fileId: 'fff7f49d-254b-478e-ab7c-f4587927cdbb', targetLocation: 'config/database.yml', variable: 'DATABASE_FILE')]){}

		// 重新建立测试用数据库
		sh "rvm ${rubyVersion} do bundle exec rake db:drop db:create db:migrate RAILS_ENV=${rails_env}"
	}
}
```

下面是测试步骤的pipeline

```groovy
node {
	def rails_env = 'test'
	stage('Preparation') {
		// ...
	}
	stage('Test') {
		def rubyVersion = sh(script: 'cat .ruby-version', returnStdout: true).trim()
		sh "rvm ${rubyVersion} do bundle exec rake test RAILS_ENV=${rails_env}"
	}
}
```

本例子测试步骤比较简单，这点因项目测试使用的工具不同而不同。至此跟上述freestyle project相同功能的pipeline script编写完成。

#### Pipeline script from SCM

这是官方推荐的使用方式，适合为一个项目的每个分支配置不同的pipeline，例如测试分支在commit后进行测试并部署到测试服务器，发布分支在commit后直接部署到生产服务器。编写方式跟Pipeline script大体相同，唯一不同的只有拉取代码的配置。当选择了`Pipeline script from SCM`后，可以配置SCM和脚本的文件名，默认为`Jenkinsfile`。 

![pipeline_from_scm]({{ sites }}/assets/2017-03-20/pipeline_from_scm.png)

然后脚本要作如下修改：

```groovy
node {
	// 从设置好的SCM拉取代码到workspace
	checkout scm
	def rails_env = 'test'
	def rubyVersion = sh(script: 'cat .ruby-version', returnStdout: true).trim()

	stage('Preparation'){
		// 此处删去拉取代码的方法
		
		sh 'source $HOME/.bashrc'
		//...
	}
	stage('Test'){
		//...
	}
}
```

## 小结

写下来感觉篇幅有点长，所以还是分为两部分吧，本篇主要介绍了Jenkins的入门使用方法，参考提供的脚本，应该可以很轻松的配置一个简单的CI工作流。下一篇将介绍一个实际项目所使用的CI工作流，同时还会介绍一些Jenkins插件。

## 参考链接

* https://mattbrictson.com/rails-continuous-integration
* https://jenkins.io/doc/