# NPM命令行工具的发布和使用

## 前言

在项目中我们经常会用到一些Node脚本命令来简化开发的工作量，例如[Element](https://github.com/ElemeFE/element)的`package.json`的[npm 脚本](https://docs.npmjs.com/misc/scripts.html)命令`build:file`(使用脚本生成Webpack的入口源文件)。简单的Node脚本命令可以解决当务之急，但是如果该Node脚本实现的功能相对复杂繁多，并且多人使用同一个脚手架生成的多个项目里有相同的Node脚本，则可能会产生以下问题：

- 对于长期的维护而言会变得相对困难
- 对于复杂的Node脚本，很难做到精准的版本控制，导致开发上沟通成本增加
- 将Node脚本植入脚手架对应的文件夹从而污染了原始文件夹是一件比较难以容忍的事情
- 如果Node脚本本身需要一些额外的依赖，那么脚手架还需要实时维护这些依赖的版本，导致脚手架的依赖增加


为了解决Node脚本命令产生的上述问题，可以采用NPM发布命令行工具（具体可查看[Creating and publishing unscoped public packages](https://docs.npmjs.com/creating-and-publishing-unscoped-public-packages)）的形式实现Node脚本命令的功能。

## NPM包管理器

NPM包管理器允许用户将自己编写的包或命令行程序上传到NPM服务器供别人使用，开发人员只需要从NPM服务器下载并安装该命令行程序到本地即可使用。同时命令行工具可以通过更新NPM服务器上的命令行程序版本进行远程升级，开发人员可以再次通过NPM服务器更新命令行工具的版本进行工具升级。

发布的命令行工具可以实现本地项目依赖安装和全局安装两种方式。采用本地项目依赖安装仍然会将发布的命令行工具加入开发态依赖列表，这不能解决项目依赖增加的问题，如果需要解决该问题，可以采用全局安装，从而将NPM命令行工具的命令链接到全局执行环境（此时Node脚本不会植入到项目中，而是存放在操作系统的用户文件夹中），实现工具命令的全局使用。


## 发布命令行工具

### 注册NPM账号

在发布命令行工具之前，需要在https://www.npmjs.com官方网站注册账号，注册成功后，需要在命令终端中使用`npm login`链接你注册的账号（`npm login`会将账号登录的证书信息保存在本地电脑，从而不需要再次登录账号），同时会在npm的网站中生成你当前登录的token信息，登录后可以通过`npm whoami`命令查看当前登录账号名。


图1 `npm login`登录
图2 `npm login`后在npm官网产生token信息


需要注意登录的时候不要使用NPM淘宝镜像地址，需要使用NPM官方地址，可以通过`npm config set registry https://registry.npmjs.org/`命令设置成NPM官方的包发布地址。


### 命令行工具文件构建


首先初始化NPM命令行工具的项目文件，使用`npm init`命令创建`package.json`文件，创建后的`package.json`包含**项目名称**、**版本**、**描述**、**项目入口文件**（在发布命令行工具时并不需要`main`字段信息，该信息主要用于发布依赖包而不是命令行程序）、**作者信息**等。


图3 `npm init`执行命令展示

其次NPM命令行工具需要配置在PATH路径下的可执行文件，在`package.json`里配置一个[`bin`](https://www.npmjs.com.cn/files/package.json/#bin)属性（具体查看https://www.npmjs.com.cn/files/package.json/#bin）属性，该属性对应的是可执行文件的路径（当使用`npm link`或者全局安装命令行工具时，NPM会为`bin`中配置的文件在`bin`目录下创建一个软连接，对于Windows系统，默认会在`C:\Users\{username}\AppData\Roaming\npm`目录下，若是局部安装则会在项目的`./node_modules/.bin`目录下创建一个软连接）。


图4 完整的package.json文件示例

配置`bin`属性的模块路径后，可以开始设计可执行文件，需要注意的是为了使入口文件使用Node作为解释程序，需要在入口文件的开始第一行写入`#! /usr/bin/env node`，目的是使用env来找到Node（在env中规定很多系统的环境变量，包括我们安装的一些环境的路径等。在不同的操作系统中，我们安装Node的路径可能会有所不同，但是其环境变量会存在于env里面，这里我们使用env来找到Node，并用Node作为解释程序。所以，env的主要目的就是让我们的脚本在不同的操作系统上都能够正常的被解释和启动），并使用Node来作为程序的解释程序执行脚本。创建的package.json文件和Node脚本如图所示。


图5 写入一行Node脚本


### NPM软链接


在命令发布前，可以通过`npm link`命令将NPM命令行工具链接到全局执行环境，从而在任意位置使用命令行都可以直接运行该NPM命令行工具。

图6 `npm link`链接


当执行`npm link`后，可以看到在Windows平台下该命令主要做了两件事，第一件事是为NPM命令行工具所在代码目录(项目的具体路径，例如在以上示例中放在了用户桌面的`npm-demo`文件目录中)创建一个软链接，将其链接到`C:\Users\{username}\AppData\Roaming\npm\node_modules\{package}`，第二件事是为NPM命令行工具的可执行文件（`package.json`文件的`bin`属性所配置的可执行文件目录）创建一个软连接，将其连接到`C:\Users\{username}\AppData\Roaming\npm\<name>`，因此在全局环境执行`<name>`（`pageage.json`中的name字段，默认如果`bin`不配置执行的命令名称时，使用`name`字段作为命令，在图4-3中执行命令的名称配置为`npm-demo`）命令时，会启用Node去执行`package.json`中`bin`字段对应的可执行文件，如图4-6所示：


图7  <name>命令执行
图8 NPM命令行工具在NPM中的软链接


如图4-7是生成的NPM软链接目录，而图4-8中产生的npm-demo文件和npm-demo.cmd文件分别是在类似于git-bash的工具中以及Windows的CMD中执行npm-demo命令时运行的脚本。

图9 NPM中生成的Shell脚本和Cmd脚本


### NPM命令行工具发布

使用npm link链接到全局环境成功后，说明命令行工具构建成功，此时可以发布命令行工具供别人下载使用，使用npm publish命令发布命令行工具（因为这里已经有人发布了npm-demo的包名称，因此采用npm-demo-ziyi2包名），如图4-9所示：

图10 `npm publish`发布命令行工具

此时查看NPM官网中的个人账号信息，可以发现发布了该命令行工具的1.0.0版本，如图4-10所示：

图11 包信息


| 依赖名称 |     描述 |   
| :-------- | :--------| 
| commander |   一个帮助快速开发nodejs命令行工具的依赖 | 
| chalk |   终端输出时颜色样式输出工具 | 
| babel-register|   对node环境中require命令加载的文件进行babel转码 | 



### 命令行工具下载安装

包的发布，主要是为了解决2中所描述的各种问题。所有开发者可以通过使用npm install  –g对   进行全局安装（演示的demo也可以通过npm install npm-demo-ziyi2 –g进行全局安装），如图4-11和4-12所示。

图12 安装成功信息

由图4-11和4-12中的安装打印信息可以发现，最终命令行工具的软链接指向了C:users\{username}\AppData\Roaming\npm\node_modules\<pageage>下的可执行脚步文件。


### 命令行工具的使用

命令行工具包安装成功之后，可以在项目中使用指定的命令行工具，如图4-13所示是

图13 命令执行


### 总结

‘1’-i18n命令行包如果有功能需要扩展，可以再次通过npm publish命令发布新版本的NPM包，此时其他开发者通过npm install全局安装的形式更新新发布版本的包，因此使用发布NPM包的形式可以使命令工具的版本升级和维护更方便。

### 参考文献


- [npm-scripts](https://docs.npmjs.com/misc/scripts)
- [npm-run-script](https://docs.npmjs.com/cli/run-script)
- [Working with package.json](https://docs.npmjs.com/getting-started/using-a-package.json)
- [npm bin](https://www.npmjs.com.cn/files/package.json/#bin)
- [How to Install Global Packages](https://docs.npmjs.com/getting-started/installing-npm-packages-globally)
- [How to Update Global Packages](https://docs.npmjs.com/getting-started/updating-global-packages)
- [How to Publish & Update a Package](https://docs.npmjs.com/getting-started/publishing-npm-packages)
- [@babel/cli](http://babeljs.io/docs/en/babel-cli/)
- [@babel/node](https://babeljs.io/docs/en/babel-node)
