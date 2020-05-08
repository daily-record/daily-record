---
title: 基于Travis的hexo自动部署
author: 霜雨
top: false
cover: true
coverImg: false
toc: true
mathjax: false
date: 2020-05-07 21:15:22
img:
password:
summary:
categories:
tags:
  - CI/CD
  - Travis
  - hexo
---

## 开始之前

首先需要能够正常部署hexo到github，这个有许多的教程，hexo的官方文档就够用了。

弄 `Travis CI` 持续集成就是为了方便写东西，在哪儿都能写，自动部署好，同时也想借这个机会了解一下 `Travis` 的持续集成是咋样的。


## `Travis CI` 是什么

`Travis CI` 提供的是持续集成（Continuous Integration）服务，它可以绑定github上的项目，只要有新的 `commit`，就会自动检测到新变化，提供一个虚拟的运行环境（一次性，每次都会重新创建环境进行构建），根据用户的配置文件 `.travis.yml` 在虚拟机中创建指定的运行环境，执行用户定义的所有命令及构建规则，并运行测试，若通过，也会按照用户定义的规则将新代码集成到主干上。


#### `.tavis.yml`

`Travis CI` 要求项目的根目录里，必须有一个 `.travis.yml` 文件。这个文件指定了 `Travis` 的行为。一旦仓库有了新的 `commit`，就会根据用户定义的工作流程执行相应的命令。

`Travis` 默认提供的运行环境，参考[可运行语言](https://docs.travis-ci.com/user/languages)。部署方式参考[Deployment](https://docs.travis-ci.com/user/deployment/)。

更详细的文件配置查看官方教程[travis tutorial](https://docs.travis-ci.com/user/tutorial/)

#### 运行流程

任何项目都会经过两个阶段：
- `install`: 安装依赖
- `script`: 运行脚本

##### `install` 字段

`install` 用来指定安装脚本

```yml
install: ./install.sh

# 如果有多个脚本，写成如下的形式
install:
  - command1
  - command2
```

上面的代码中，如果 `command1` 失败了，整个构建就会停下来，不再往下进行。

如果不需要安装，即跳过安装阶段，就直接设为 `true`。

```yml
install: true
```

##### `script` 字段

`script` 用来指定构建的命令及运行的脚本，包含测试脚本。

```yml
script: bundle exec thor build

# 如果有多个脚本，写成如下的形式
install:
  - command1
  - command2
```

**注意，`script` 与 `install` 字段不一样，如果 `command1` 失败 ，`command2` 会继续执行。但是整个构建阶段的状态依然是失败。**

如果想让命令顺序执行，则要写成如下形式：

```yml
script: command1 && command2
```

##### `deploy` 字段

`script` 阶段结束后，可以设置[通知步骤](https://docs.travis-ci.com/user/notifications/)和[部署](https://docs.travis-ci.com/user/deployment/)，但并不是必须的。

部署的脚本可以在 `script` 阶段执行，也可以使用 `Travis` 为几十种常见服务提供的快捷部署功能。比如部署到 `github pages`，可以使用 `provider: pages` 字段：

```yml
deploy:
  provider: pages
  skip_cleanup: true
  github_token: $GITHUB_TOKEN # Set in travis-ci.org dashboard
  on:
    branch: master
```

更多的部署方式，可以查看[官方Deplyment文档](https://docs.travis-ci.com/user/deployment/)。

##### 钩子方法 

`Travis` 为上面这些阶段提供了7个钩子：

- `before_install`: install 阶段之前执行
- `before_script`: script 阶段之前执行
- `after_failure`: script 阶段失败时执行
- `after_success`: script 阶段成功时执行
- `before_deploy`: deploy 步骤之前执行
- `after_deploy`: deploy 步骤之后执行
- `after_script`: script 阶段之后执行

完整的生命周期，从开始到结束是下面的流程：

1. `before_install`
2. `install`
3. `before_script`
4. `script`
5. `aftersuccess or afterfailure`
6. `[OPTIONAL] before_deploy`
7. `[OPTIONAL] deploy`
8. `[OPTIONAL] after_deploy`
9. `after_script`


##### 运行状态

`Travis` 每次运行，可能会返回四种状态：

- `passed`: 运行成功，所有步骤的退出码都是0
- `canceled`: 用户取消执行
- `errored`: `before_install`、`install`、`before_script` 有非零退出码，运行会立即停止
- `failed`: `script` 有非零状态码 ，会继续运行


## 操作流程

#### 设置 `Travis CI` 的仓库访问权限

使用github账号登录 `Travis CI`，此时会让你在github中安装 `Travis CI`， 进入github，一步步打开 `profile_image -> settings -> Applications`，在此处可以看到应用，安装并配置仓库的权限，我给的权限是 `All repositories`，使其可以访问所有的仓库。

![](https://raw.githubusercontent.com/daily-record/daily-record-img/master/20200507222016.png)

![](https://raw.githubusercontent.com/daily-record/daily-record-img/master/20200507222120.png)

这是 `Travis CI` 当中就可以访问所有的仓库了，不过这时它只能访问仓库，还需要进一步配置才能完成自动化部署。


#### `Travis CI` 环境变量

在 `Travis CI` 中可以通过设置环境变量来传递一些不便于写在配置文件中的值（如密码、密钥之类的）。

由于构建环境需要对应仓库的权限，所以要使用 `GitHub Personal Access Token`，在命令行中执行git操作用其代替密码（当然，也可以使用密码，只不过更改密码后 `Travis CI` 无法完成自动构建任务），可以在github中获取 `token` 并将其作为变量存入 `Travis CI` 中，这样就可以直接在配置文件中调用此变量进行使用，由于只有 `Travis Ci` 有变量的值，所以安全性还是有保障的。


##### 获取 `GitHub Personal Access Token`

一步步打开 `profile_image -> settings -> Developer -> Personal Access Token`，生成一个新的 `token`。由于只需对仓库进行push操作，所以仅把勾上 `repo` 部分勾上。

![](https://raw.githubusercontent.com/daily-record/daily-record-img/master/20200508004800.png)

生成的 `token` 只显示一次，所以要及时保存下来。


##### 设置 `Travis CI` 环境变量

打开 `Travis CI`，进入需要操作的 `repo` 的settings，添加一个环境变量 `GITHUB_TOKEN`，将上一步骤中的 `token` 填入，`Branch` 选择需要进行构建的源代码分支（All branches也可）。

![](https://raw.githubusercontent.com/daily-record/daily-record-img/master/20200508005556.png)


#### 配置 `.travis.yml` 文件

在上面所有操作过后，最后剩下的就是利用配置文件指定 `Travis CI` 的行为。

##### 写入部署环境

指定需要使用的语言及对应的版本。

```yml
sudo: false
language: node_js
node_js:
  - 12 # use nodejs v12 LTS
```

##### 缓存 `node_modules` 目录

`node_modules` 里包含了很多模块，每次重新下载安装花费大量时间，于是缓存这个目录。

```yml
cache:
  directories:
    - node_modules
```

##### 指定操作的分支

由于repo的 `master` 分支只能放hexo生成的静态文件(public文件夹)，所以需要将hexo的源目录丢在其它分支上，我使用 `hexo` 分支存储这些文件，所以只需要让 `Travis` 检测 `hexo` 分支的变动就行。

```yml
# assign build branches
branches:
  only:
    - hexo
```

##### 构建

这个按照hexo生成的顺序写入命令即可，可以在 `Travis` 的虚拟环境中正常生成所有静态文件

```yml
before_install:
  - npm install -g hexo-server
  - npm install -g hexo-cli # install hexo
  - npm install -g hexo
  - npm install hexo-deployer-git --save

install:
  - npm install 

script:
  - hexo generate # generate static files
```


##### 部署

使用 `Travis` 提供的快捷部署，这样就不需要自己写入各种git命令push到仓库中。

详细配置参考官方[GitHub Pages Deployment](https://docs.travis-ci.com/user/deployment/pages/)。

以下参数大概的意思就是使用github的快捷部署，将生成的文件夹 `local-dir: public` push到github的 `target_branch: master` 分支。

```yml
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GITHUB_TOKEN
  keep-history: true
  on:
  #   branch: master
    all_branches: true # solve a permission problem
  target_branch: master
  local-dir: public # Directory to push to GitHub Pages, defaults to current directory.
                    # Can be specified as an absolute path or a relative path from the current directory.
```

#### 完整配置文件

```yml
sudo: false
language: node_js
node_js:
  - 12 # use nodejs v12 LTS
cache:
  directories:
    - node_modules
# assign build branches
branches:
  only:
    - hexo

before_install:
  - npm install -g hexo-server
  - npm install -g hexo-cli # install hexo
  - npm install -g hexo
  - npm install hexo-deployer-git --save

install:
  - npm install 

script:
  - hexo generate # generate static files

deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GITHUB_TOKEN
  keep-history: true
  on:
  #   branch: master
    all_branches: true # solve a permission problem
  target_branch: master
  local-dir: public # Directory to push to GitHub Pages, defaults to current directory.
                    # Can be specified as an absolute path or a relative path from the current directory.
```