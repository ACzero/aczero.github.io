---
layout: post
title:  "为Rails项目搭建Jenkins CI服务器(二)"
date:   2017-03-21 22:30:00 +0800
categories: other
---

## 前言

本篇续上一篇<<为Rails项目搭建Jenkins CI服务器(一)>>，将介绍一个实际使用的CI流程。

## Bitbucket+Pipeline实现rails项目CI服务

首先介绍这个CI服务的工作流，先上图：

 ![ci_workflow]({{ sites }}/assets/2017-03-22/ci_workflow.png)

 当bitbucket上有新的commit或者pull request时，会通知Jenkins服务器，CI工作流开始。首先是`Preparation`阶段，拉取代码后执行bundle install等命令，为rails项目准备测试环境。接着是`Test`阶段，运行项目对应的测试，输出测试结果。接着是`Report`阶段，可以使用`brakeman`，`rubocop`和`simplecov`等项目检测工具，生成HTML页面报告。最后检测当前分支，若是staging用的分支则部署到staging环境，其他分支则跳过。最后将CI结果返回给bitbucket，要注意的是，只要其中某个stage出错，就会直接返回CI结果。 

在Jenkins后台可以查看生成的HTML报告：

 ![html_reports]({{ sites }}/assets/2017-03-22/html_reports.png)

在bitbucket每个commit可以查看build结果：

 ![bitbucket_commit]({{ sites }}/assets/2017-03-22/bitbucket_commit.png)

### 项目具体配置

接下来介绍配置步骤。使用`Multibranch Pipeline`类型的项目，这种项目能跟踪整个repo的提交。创建成功后，必须配置的只有`Branch Sources`，使用bitbucket的话需要安装[Bitbucket Plugin](https://wiki.jenkins-ci.org/display/JENKINS/BitBucket+Plugin)。配置如图：

 ![ci_workflow]({{ sites }}/assets/2017-03-22/multibranch.png)

需要注意的是，bitbucket的`Scan Credentials`仅支持使用bitbucket账号，建议使用一个只有读权限的账号。相应填入`owner`和`Repoitory Name`信息，`Property strategy`可以指定哪些branch不执行ci。最后说一下`Auto-register webhook`这个checkbox，Jenkins配置完成后，用户还需要在bitbucket配置webhook通知Jenkins，这个配置需要项目管理员权限，因此建议不要勾选这个checkbox而是手动去配置。文档给出的配置要点为：

* URL: [JENKINS_ROOT_URL]/bitbucket-scmsource-hook/notify
* Check "Push", "Pull Request Created" and "Pull Request Updated" in the triggers section.

该配置在项目首页 -> Settings -> Webhooks。创建一个webhook，url按上面指示填写，Triggers选择`Choose from a full list of triggers`然后按指示勾选即可。

完成以上步骤后项目配置就完成了，对于其他SCM，请根据文档配置。

### pipeline脚本

直接贴出Jenkinsfile，具体解释在脚本的注释部分

```groovy
// plugins:
// SSH Agent Plugin
// Config File Provider Plugin
// HTML Publisher plugin

node {
  checkout scm
  def rails_env = 'test'
  // 获取项目的ruby版本
  def rubyVersion = sh(script: 'cat .ruby-version', returnStdout: true).trim()

  stage('Preparation'){
    // 若提示command not found等错误，检查shell配置:https://issues.jenkins-ci.org/browse/JENKINS-29877
    sh 'source $HOME/.bashrc'
    
    // 安装对应版本的ruby
    sh """if ! rvm ${rubyVersion} do ruby -v &> /dev/null; then
    rvm get stable
    rvm install ${rubyVersion}
    fi"""

    // 安装对应bundler
    def bundlerVersion = sh(script: '''rvm all do ruby -e 'puts $<.read[/BUNDLED WITH\\n   (\\S+)$/, 1] || "<1.10"' Gemfile.lock''', returnStdout: true).trim()
    sh "rvm ${rubyVersion} do gem install bundler --conservative --no-document -v ${bundlerVersion}"
    // --deployment http://bundler.io/deploying.html#deploying-your-application
    sh "rvm ${rubyVersion} do bundle install --deployment --retry=3"

    // 复制配置文件，使用Config File Provider Plugin
    configFileProvider([configFile(fileId: 'cd003fbc-d1ae-496b-b68c-b08f0640a286', targetLocation: 'config/secrets.yml', variable: 'SECRET_FILE'), configFile(fileId: 'fff7f49d-254b-478e-ab7c-f4587927cdbb', targetLocation: 'config/database.yml', variable: 'DATABASE_FILE')]){
    }

    // 重建数据库
    sh "rvm ${rubyVersion} do bundle exec rake db:drop db:create db:migrate RAILS_ENV=${rails_env}"
  }
  stage('Test'){
    sh "rvm ${rubyVersion} do bundle exec rake test RAILS_ENV=${rails_env}"
  }
  stage('Report'){
    // 生成brakeman和rubocop报告，以HTML输出
    sh "rvm ${rubyVersion} do bundle exec brakeman --summary -o brakeman.html"
    sh "rvm ${rubyVersion} do bundle exec rubocop --out rubocop.html || true"
    
    // 使用HTML Publisher plugin，在项目页面生成HTML展示页面链接。
    publishHTML([allowMissing: true, alwaysLinkToLastBuild: false, keepAll: false, reportDir: './', reportFiles: 'brakeman.html', reportName: 'Brake Report'])
    publishHTML([allowMissing: true, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'coverage/', reportFiles: 'index.html', reportName: 'SimpleCov Report'])
    publishHTML([allowMissing: true, alwaysLinkToLastBuild: false, keepAll: false, reportDir: './', reportFiles: 'rubocop.html', reportName: 'Rubocop Report'])
  }
  if(env.BRANCH_NAME=='master'){
  	stage('Deploy'){
      // 使用SSH Agent Plugin提供ssh key，用capistrano进行staging环境的部署
      sshagent(['b33c1c57-737d-4195-a6f5-446a9d000f2a']) {
        sh "rvm ${rubyVersion} do bundle exec cap staging deploy"
      }
    }
  }else{
    println 'not on master, skip deploy'
  }
}
```

`Preparation`和`Test`部分在之前的pipeline介绍中已经解释过。`Report`部分可以使用一些项目检测工具如[Rubocop](https://github.com/bbatsov/rubocop)，[Simplecov](https://github.com/colszowka/simplecov)和[Brakeman](https://github.com/presidentbeef/brakeman)等输出html页面。在pipeline中没有去生成simplecov的报告，因为在测试的阶段就会生成。然后使用[HTML Publisher plugin](https://wiki.jenkins-ci.org/display/JENKINS/HTML+Publisher+Plugin)发布页面。

### 注意事项

使用HTML Publisher plugin需要修改Jenkins默认的[CSP](https://en.wikipedia.org/wiki/Content_Security_Policy)配置(会产生安全问题，在不了解的情况下不建议修改)，否则发布出来的页面的css和js都会被禁用。修改的方法查看[官方文档](https://wiki.jenkins-ci.org/display/JENKINS/Configuring+Content+Security+Policy)，具体如何配置要根据发布的页面对应设置。设置可以在Manage Jenkins -> Script Console中执行脚本来设置，但在Jenkins重启后将会重新从配置文件加载，所以想保留修改的话要在`/etc/default/jenkins`中作为参数传递。像这样修改JAVA_ARGS参数：

```
JAVA_ARGS="-Djava.awt.headless=true -Dhudson.model.DirectoryBrowserSupport.CSP=\"default-src 'self'; style-src 'self' 'unsafe-inline'; script-src 'self' 'unsafe-eval'; img-src 'self'\""
```

## 总结

看到这里，我想你应该已经掌握了Jenkins的使用方法。使用下来个人感觉Jenkins是一个很完善的自动化工具，你可以用Jenkins来实现CI,CD等工作流。Jenkins还有非常多的插件，因此可以快速实现你的工作流，对于Github，Bitbucket都有比较完善的支持。Pipeline的引入又提供了极大的自由度，折腾起来还是非常有意思的 。但Jenkins有一部分插件的文档不完善，而官方的文档也是略为简略，所以坑还是有不少的。对于Jenkins，我也只有配置简单工作流的经验，欢迎指出文章中的不足之处，也欢迎一起交流下使用心得。

## 参考链接

* https://mattbrictson.com/rails-continuous-integration
* https://wiki.jenkins-ci.org/display/JENKINS/Configuring+Content+Security+Policy
* https://go.cloudbees.com/docs/cloudbees-documentation/cje-user-guide/index.html