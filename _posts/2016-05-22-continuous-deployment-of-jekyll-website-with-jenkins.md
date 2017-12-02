---
layout: post
title:  "Continuous deployment of Jekyll website with Jenkins"
date:   2016-05-22 00:00:00
categories: jekyll jenkins rake bundler docker ruby devops
image-base: /assets/images/posts/2016-05-22-continuous-deployment-of-jekyll-website-with-jenkins
---

A couple of days ago any update to my website was followed by a quick flurry of manual actions to build and upload the
changes. All of this was a result of misplaced laziness, as I already had Jenkins setup for a bunch of .NET
projects. However, I've finally got around to creating a build job for my [Jekyll](http://jekyllrb.com/) website...

![Stage view of Jenkins job]({{ page.image-base }}/job-stage-view.png)

## Preparation

As mentioned, my website is created using [Jekyll](http://jekyllrb.com/), a static-site generator
written in Ruby. So it made sense to use a [Gemfile](http://bundler.io/v1.5/gemfile.html) to define the dependencies and
[rake](https://github.com/ruby/rake) to manage the automated build tasks for the Jenkins job.

The main tasks for the job are to build the website and to test the generated HTML. Building the website was a simple
case of wrapping Jekyll's commands, and equally as simple was testing the generated HTML, which I did using
[HTML Proofer](https://github.com/gjtorikian/html-proofer).

```ruby
require 'html-proofer'

$sourceDir = "./source"
$outputDir = "./output"
$testOpts = {
  # Ignore errors "linking to internal hash # that does not exist"
  :url_ignore => ["#"],
  # Allow empty alt tags (e.g. alt="") as these represent presentational images
  :empty_alt_ignore => true
}

task :default => ["serve:development"]

desc "cleans the output directory"
task :clean do
  sh "jekyll clean"
end

namespace :build do

  desc "build development site"
  task :development => [:clean] do
    sh "jekyll build --drafts"
  end

  desc "build production site"
  task :production => [:clean] do
    sh "JEKYLL_ENV=production jekyll build --config=_config.yml,_config_prod.yml"
  end
end

namespace :serve do

  desc "serve development site"
  task :development => [:clean] do
    sh "jekyll serve --drafts"
  end

  desc "serve production site"
  task :production => [:clean] do
    sh "JEKYLL_ENV=production jekyll serve --config=_config.yml,_config_prod.yml"
  end
end

namespace :test do

  desc "test production build"
  task :production => ["build:production"] do
    HTMLProofer.check_directory($outputDir, $testOpts).run
  end
end
```

## Continuous deployment pipeline

The build process is defined in a `Jenkinsfile` (the pipeline-as-a-script pushed out in Jenkins v2) which resides in
my website's repository.

The build environment is created using Docker with the [ruby image](https://hub.docker.com/r/library/ruby/) which is
used in the pipeline with the
[CloudBees Docker Pipeline plugin](https://wiki.jenkins-ci.org/display/JENKINS/Docker+Pipeline+Plugin).
It's then just a case of downloading the dependencies with the `bundle` command and calling each of the rake tasks.

The last part, the deployment stage, requires a bit more work than I would have liked, due to the setup of the host's
web-server. So for the time being I'm using the
[Credentials Binding plugin](https://wiki.jenkins-ci.org/display/JENKINS/Credentials+Binding+Plugin) to inject the
credentials into both SSH to clear the server and SCP to upload the newly generated files.

```groovy
node('docker') {
  deleteDir()

  stage 'Checkout'
  checkout scm

  def buildEnv = docker.image('ruby:latest')
  buildEnv.pull()
  buildEnv.inside {
    stage 'D/L dependencies'
    sh 'bundle'

    stage 'Build'
    sh 'rake build:production'

    stage 'Test'
    try {
      sh 'rake test:production'
    } catch (err) {
      currentBuild.result = 'UNSTABLE'
    }

    stage 'Build production'
    sh 'rake build:production'
    archive 'output/**'

    stash includes: 'output/**', name: 'built-site'
  }
}

if (currentBuild.result == 'UNSTABLE') {
  echo 'Skipping deployment due to unstable build'
} else {

  stage 'Deploy'
  node() {
    deleteDir()
    unstash 'built-site'

    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'website', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME']]) {
      sh 'SSHPASS=$PASSWORD sshpass -e ssh -l $USERNAME ${env.HOST} rm -rf /var/www/vhosts/flyingtophat.co.uk/httpdocs/*'
      sh 'SSHPASS=$PASSWORD sshpass -e scp -r output/* $USERNAME@${env.HOST}:/var/www/vhosts/flyingtophat.co.uk/httpdocs'
    }
  }
}
```
