人工翻译自官方文档：https://docs.gitlab.com/ce/ci/yaml/README.html
欢迎转载，请注明出处：https://github.com/szyhf/gitlab-study

本文整理了.gitlab-ci.yml的使用方式，这个文件被GitLabRunner用于管理项目的构建。

# .gitlab-ci.yml

.gitlab-ci.yml是从7.12版本开始启用的项目CI配置文件，本文件应该放在项目的根目录中，里边描述了你的项目应该如何构建。

一个yaml文件定义了一组各不相同的job，并定义了它们应该怎么运行。这组jobs会被定义为yaml文件的顶级元素，并且每个job的子元素中总有一个名为script的节点。

> A set of jobs，强调是Set所以名称必须不同。

```yaml
job1:
	script: "job1的执行脚本命令（shell）“

job2:
	script:
		- "job2的脚本命令1（shell)"
		- “job2的脚本命令2（shell）”
```

上述的文件是最简单的CI配置例子，每个job都执行了不同的命令，其中job1只执行了1条命令，job2通过数组的定义按顺序执行了两条命令。

当然，每个命令可以直接执行代码（`./configure;make;make install`）或者在仓库目中运行另一个脚本（`test.sh`）。

Job配置会被Runner读取并用于构建项目，并且在Runner的环境中被执行。很重要的一点是，每个job都会独立的运行，相互间并不依赖。

> Each job is run independently from each other.

下边是一个比较复杂的CI配置文件例子：

```yaml
image: ruby:2.1
services:
  - postgres

before_script:
  - bundle install

after_script:
  - rm secrets

stages:
  - build
  - test
  - deploy

job1:
  stage: build
  script:
    - execute-script-for-job1
  only:
    - master
  tags:
    - docker
```

以下是一些保留字，这些单词不能被用于命名job：

|保留字|必填|介绍|
|-|-|-|
|image|否|构建使用的Docker镜像名称，[使用Docker](https://docs.gitlab.com/ce/ci/docker/README.html) )作为Excutor时有效}
|services|否|使用的Docker服务，[使用Docker](https://docs.gitlab.com/ce/ci/docker/README.html) 作为Excutor时有效|
|stages|否|定义构建的stages|
|types|否|`stages`的别名|
|before_script|否|定义所有job执行之前需要执行的脚本命令|
|after_script|否|定义所有job执行完成后需要执行的脚本命令|
|variables|否|定义构建变量|
|cache|否|定义一组文件，该组文件会在运行时被缓存，下次运行仍然可以使用|

## image和services

这两个关键字允许用户自定义运行时使用的Docker image，及一组可以在构建时使用的Service。这个特性的详细说明在 [Docker integration - GitLab Documentation](https://docs.gitlab.com/ce/ci/docker/README.html)中。

## before_script

被用于定义所有job被执行之前的命令，包括部署构建环境。它可以是一个数组元素或者一个多行文本元素。

## after_script

> Gitlab8.7及GitlabRunner1.2以上才支持本关键字

被用于定义所有job被执行完之后要执行的命令。同样可以是一个数组或者一个多行文本。

## stages

用于定义可以被job使用的stage. 定义stages可以实现柔性的多stage执行管道。

> The specification of stages allows for having flexible multi stage pipelines.

stages定义的元素顺序决定了构建的执行顺序：

1. 同样stage的job是并行执行的。
2. 下一个stage的jobs是当上一个stage的josb全部执行成功后才会执行。

让我们看一个简单的例子，以下有3个stage：

```yaml
stages:
	- build
	- test
	- deploy
```

1. 首先，所有stage属性为build的job会被并行执行.
2. 如果所有stage属性为build的job都执行成功了，stage为test的job会被并行执行。
3. 如果所有stage为test的job都执行成功了，则stage为deploy的job会被并行执行。
4. 如果所有stage为deploy的job都执行成功了，则提交被标记为success。
5. 如果任何一个前置job失败了，则提交被标记为failed并且任何下一个stage的job都不会被执行。

两个有价值的提示：

1. 如果配置文件中没有定义stages，那么默认情况下的stages属性为build、test和deploy。
2. 如果一个job没有定义stage属性，则它的stage属性默认为test。

## types

stages的别名。

## variables

> 从Runner0.5.0开始支持

GitlabCI允许你在`.gitlab-ci.yml`文件中设置构建环境的环境变量。这些变量会被存储在git仓库中并用于记录不敏感的项目配置信息，例如：

```yaml
variables:
  DATABASE_URL: "postgres://postgres@postgres/my_database"
```

这些变量会在之后被用于执行所有的命令和脚本。Yaml配置的变量同样会被设置于所有被建立的service容器中，这可以让使用更加方便。variables同样可以设置为job级别的属性。

除了用户自定义的变量外，同样有Runner自动设置的变量。例如说`CI_BUILD_REF_NAME`，这个变量定义了正在构建的git仓库的branch或者tag的名称。

除了在`.gitlab-ci.yml`中设置的非敏感变量外，Gitlab的UI中还提供了设置敏感变量的功能。

[更多关于变量的说明](https://docs.gitlab.com/ce/ci/variables/README.html)

cache

> Runner0.7.0之后被介绍

用于定义一系列需要在构建时被缓存的文件或者目录。只能定义在项目工作环境中的目录或者文件。

> cache is used to specify a list of files and directories which should be cached between builds.

默认情况下缓存功能是对每个job和每个branc都开启的。

如果`cache`在job元素之外被蒂尼，这意味着全局设置，并且所有的job会使用这个设置。以下是一些例子：

### 缓存所有在`binaries`目录中的文件和`.config`文件：

```yaml
rspec:
  script: test
  cache:
    paths:
    - binaries/
    - .config
```

### 缓存所有git未追踪的文件

```yaml
rspec:
  script: test
  cache:
    untracked: true
```

### 缓存所有git未追踪的文件和在binaries目录下的文件

```yaml
rspec:
  script: test
  cache:
    untracked: true
    paths:
    - binaries/
```

### job级别定义的cache设置会负载全局级别的cache配置。该例子将仅缓存目录`binaries`

```yaml
cache:
  paths:
  - my/files

rspec:
  script: test
  cache:
    paths:
    - binaries/
```

cache功能仅提供最努力的支持，但不要指望它总能生效。更多的实现细节，可以参阅GitlabRunner。

> The cache is provided on a best-effort basis, so don't expect that the cache will be always present.

## cache:key

> Runner1.0.0中被介绍

key允许你在不同的job之间定义cache的种类，例如所有job共享的cache单例、一个job一个的cache、一个branch一个的cache等等。

这允许你更便利的使用缓存，允许你在不同的job甚至不同的branche间共享缓存。

`cache:key`变量可以使用任何之前定义的变量。

### 例子
per-job caching:
```yaml
cache:
  key: "$CI_BUILD_NAME"
  untracked: true
```
per-branch caching:
```yaml
cache:
  key: "$CI_BUILD_REF_NAME"
  untracked: true
```
per-job and per-branch caching:
```yaml
cache:
  key: "$CI_BUILD_NAME/$CI_BUILD_REF_NAME"
  untracked: true
```
per-branch and per-stage caching:
```yaml
cache:
  key: "$CI_BUILD_STAGE/$CI_BUILD_REF_NAME"
  untracked: true
```

> 这段感觉怎么翻译都啰嗦，就这样吧

如果是在Windows环境下开发，需要使用%代替$标识环境变量：

```yaml
cache:
  key: "%CI_BUILD_STAGE%/%CI_BUILD_REF_NAME%"
  untracked: true
```

(未完待续)