# NPM发布和使用命令行(CLI)工具

## 前言

在项目中我们经常会用到一些Node脚本命令来简化开发的工作量，例如[Element](https://github.com/ElemeFE/element)的`package.json`的[npm 脚本](https://docs.npmjs.com/misc/scripts.html)命令`build:file`(使用脚本生成Webpack的入口源文件)。简单的Node脚本命令可以解决当务之急，但是如果该Node脚本实现的功能相对复杂繁多，并且多人使用同一个脚手架生成的多个项目里有相同的Node脚本，则可能会产生以下问题：

- 对于长期的维护而言会变得相对困难
- 对于复杂的Node脚本，很难做到精准的版本控制，导致开发上沟通成本增加
- 将Node脚本植入脚手架对应的文件夹从而污染了原始文件夹是一件比较难以容忍的事情
- 如果Node脚本本身需要一些额外的依赖，那么脚手架还需要实时维护这些依赖的版本，导致脚手架可见的依赖增加


为了解决Node脚本命令产生的上述问题，可以采用NPM发布命令行（CLI）工具（具体可查看[Creating and publishing unscoped public packages](https://docs.npmjs.com/creating-and-publishing-unscoped-public-packages)）的形式实现Node脚本命令的功能。

## NPM包管理器

NPM包管理器允许用户将自己编写的包或命令行工具上传到NPM服务器，开发人员只需要从NPM服务器下载并安装该命令行工具到本地即可使用。命令行工具可以通过更新NPM服务器上的命令行程序版本进行远程升级，开发人员可以再次通过NPM服务器更新命令行工具的版本进行工具升级。

发布的命令行工具可以实现本地项目依赖安装和全局安装两种方式。采用本地项目依赖安装会将发布的命令行工具加入到项目的开发态依赖列表，这不能解决项目依赖增加的问题，如果需要解决该问题，可以采用全局方式安装，从而将NPM命令行工具的命令链接到全局执行环境（此时Node脚本不会植入到项目中，而是存放在操作系统的用户文件夹中），实现命令行工具的全局使用。

这里列举几个常见的命令行工具：

- [npm/cli](https://github.com/npm/cli)
- [@babel/cli](https://babeljs.io/docs/en/next/babel-cli.html)
- [vuejs/vue-cli](https://github.com/vuejs/vue-cli) 
- [commitizen/cz-cli](https://github.com/commitizen/cz-cli) 


## 构建、发布和使用命令行工具


### 构建命令行工具


首先新建NPM命令行工具的项目文件，使用`npm init`命令创建`package.json`命令行工具的描述文件，创建后的`package.json`包含**项目名称**、**版本**、**描述**、**项目入口文件**（在发布命令行工具时并不需要`main`字段信息，该信息主要用于发布依赖包而不是命令行工具）、**作者信息**等。执行`npm init`：

``` javascript
AppledeMacBook-Pro:npm-cli-package ziyi2$ npm init
This utility will walk you through creating a package.json file.
It only covers the most common items, and tries to guess sensible defaults.

See `npm help json` for definitive documentation on these fields
and exactly what they do.

Use `npm install <pkg>` afterwards to install a package and
save it as a dependency in the package.json file.

Press ^C at any time to quit.
package name: (npm-cli-package) 
version: (1.0.0) 
description: NPM命令行工具的发布和使用
entry point: (index.js) 
test command: 
git repository: (https://github.com/ziyi2/npm-cli-package.git) 
keywords: npm cli
author: ziyi2
license: (ISC) 
About to write to /Users/ziyi2/Git/npm-cli-package/package.json:

{
  "name": "npm-cli-package",
  "version": "1.0.0",
  "description": "NPM命令行工具的发布和使用",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/ziyi2/npm-cli-package.git"
  },
  "keywords": [
    "npm",
    "cli"
  ],
  "author": "ziyi2",
  "license": "ISC",
  "bugs": {
    "url": "https://github.com/ziyi2/npm-cli-package/issues"
  },
  "homepage": "https://github.com/ziyi2/npm-cli-package#readme"
}


Is this OK? (yes) 
```


执行完后生成的`package.json`描述文件如下：

``` javascript
{
  "name": "npm-cli-package",
  "version": "1.0.0",
  "description": "NPM命令行工具的发布和使用",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/ziyi2/npm-cli-package.git"
  },
  "keywords": [
    "npm",
    "cli"
  ],
  "author": "ziyi2",
  "license": "ISC",
  "bugs": {
    "url": "https://github.com/ziyi2/npm-cli-package/issues"
  },
  "homepage": "https://github.com/ziyi2/npm-cli-package#readme"
}

```

其次NPM命令行工具需要配置在PATH路径下的可执行文件，在`package.json`里配置一个[`bin`](https://www.npmjs.com.cn/files/package.json/#bin)属性，该属性对应的是可执行文件的路径（当使用`npm link`或者全局安装命令行工具时，NPM会为`bin`中配置的文件在`bin`目录下创建一个软连接，对于Windows系统，默认会在`C:\Users\{username}\AppData\Roaming\npm`目录下，若是局部安装则会在项目的`./node_modules/.bin`目录下创建一个软连接）。

例如将`bin`对应的可执行文件路径配置为当前项目下的`src/index.js`:

``` javascript
"bin": {
   // npm-cli-package是一个可执行的命令，该命令指向了`src/index.js`脚本(这里暂时还不清楚该脚本的解释器)
   "npm-cli-package": "src/index.js"
 },
```

配置`bin`属性的模块路径后，可以开始设计可执行文件。为了使入口文件使用Node作为解释程序，需要在入口文件的头部写入`#! /usr/bin/env node`，目的是使用env来寻找操作系统中的Node启动路径，并将Node作为程序的解释器（在env中包含了一些环境变量，包括我们安装的一些环境的路径等。在不同的操作系统中，我们安装Node的路径可能会有所不同，但是其环境变量会存在于env中，这里我们使用env来找到Node的执行路径。因此env的主要目的就是让我们的脚本在不同的操作系统上都能够正常的被解释和执行）。例如在`src/index.js`入口文件写入一个打印信息的Node脚本：

``` javascript
#! /usr/bin/env node
console.info('npm-cli-package：', '1.0.0')
```


### NPM软链接


命令行工具构建完成后，接下来是测试该命令行工具能否被正常使用，此时可以通过`npm link`命令将NPM命令行工具链接到全局执行环境，从而在系统的任意路径下都可以使用该NPM命令行工具。执行`npm link`：

``` javascript
// Windows
PS C:\Users\ziyi2\Desktop\npm-cli-package> npm link
up to date in 0.428s
C:\Users\ziyi2\AppData\Roaming\npm\npm-cli-package -> C:\Users\ziyi2\AppData\Roaming\npm\node_modules\npm-cli-package\src\index.js
C:\Users\ziyi2\AppData\Roaming\npm\node_modules\npm-cli-package -> C:\Users\ziyi2\Desktop\npm-cli-package

// MAC OS X
AppledeMacBook-Pro:npm-cli-package ziyi2$ npm link
npm notice created a lockfile as package-lock.json. You should commit this file.
up to date in 4.113s
found 0 vulnerabilities

/usr/local/bin/npm-cli-package -> /usr/local/lib/node_modules/npm-cli-package/src/index.js
/usr/local/lib/node_modules/npm-cli-package -> /Users/ziyi2/Git/npm-cli-package
```


当执行`npm link`后，可以看到在Mac系统下该命令主要做了两件事：

- 为NPM命令行工具的可执行文件（`package.json`文件的`bin`属性所配置的可执行文件`src/index.js`）创建一个软链接，将其链接到`/usr/local/bin/<package>`（Windows下是`C:\Users\{username}\AppData\Roaming\npm\<package>`）
- 为NPM命令行工具所在项目路径(项目的具体路径，例如在以上示例中放在了`/Users/ziyi2/Git`的`npm-cli-package`文件目录中)创建一个软链接，将其链接到`/usr/local/lib/node_modules/<package>`（Windows下是`C:\Users\{username}\AppData\Roaming\npm\node_modules\<package>`)


因此在全局环境执行`<name>`命令时（`pageage.json`中的`name`字段，默认如果`bin`不配置执行的命令名称时，使用`name`字段作为命令，这里演示中将执行的命令名称配置成了`npm-cli-package`），会启用Node去执行`package.json`中`bin`字段对应的可执行文件，此时可在任意位置执行`npm-cli-package`命令：

``` javascript
AppledeMacBook-Pro:npm-cli-package ziyi2$ npm-cli-package
npm-cli-package： 1.0.0
```

可以发现在当前项目外的任意路径都可以使用该命令成功打印信息，说明Node解释器和软链接都设置成功。Windows系统可以到用户目录`C:\Users\{username}\AppData\Roaming\npm\node_modules`下查看`npm-cli-package`包的软链接，并且可以在`C:\Users\{username}\AppData\Roaming\npm`中找到`npm-cli-package`(Shell)和`npm-cli-package.cmd`(Cmd)两个可执行文件。


在真正设计CLI工具时，你可能需要一些额外功能的依赖，例如：

- [commander.js](https://github.com/tj/commander.js) - node.js command-line interfaces made easy
- [chalk](https://github.com/chalk/chalk) - Terminal string styling done right
- [@babel/register](https://babeljs.io/docs/en/babel-register/) - 对Node.js环境中require命令加载的文件进行babel转码


### 发布NPM命令行工具

通过`npm link`以及命令行的使用测试，发现命令行工具没有任何问题，此时想将该工具分享给他人使用，可以利用NPM包管理器的发布机制。在发布命令行工具之前，需要在[NPM官网](https://www.npmjs.com)注册账号（需要设置邮箱并进行邮箱），注册成功后，在命令终端中使用`npm login`链接你注册的账号（`npm login`会将账号登录的证书信息保存在本地电脑，从而不需要再次登录账号），同时会在npm的网站中生成你当前登录的token信息，登录后可以通过`npm whoami`命令查看当前登录账号名。注意登录的时候不要使用NPM淘宝镜像地址，需要使用NPM官方地址，可以通过`npm config set registry https://registry.npmjs.org/`命令设置成NPM官方的包发布地址。。

``` javascript
AppledeMacBook-Pro:~ ziyi2$ npm login
Username: ziyi222
Password: 
Email: (this IS public) 673191402@qq.com
Logged in as ziyi222 on https://registry.npmjs.org/.
```


`npm login`后会在npm官网产生token信息。







使用`npm publish`命令发布命令行工具:

``` javascript
AppledeMacBook-Pro:npm-cli-package ziyi2$ npm-cli-package
npm-cli-package： 1.0.0
AppledeMacBook-Pro:npm-cli-package ziyi2$ npm publish
npm notice 
npm notice 📦  npm-cli-package@1.0.0
npm notice === Tarball Contents === 
npm notice 595B   package.json
npm notice 12.1kB README.md   
npm notice 64B    src/index.js
npm notice === Tarball Details === 
npm notice name:          npm-cli-package                         
npm notice version:       1.0.0                                   
npm notice package size:  5.1 kB                                  
npm notice unpacked size: 12.8 kB                                 
npm notice shasum:        624e3e45667da53d474418907d3250336b56208b
npm notice integrity:     sha512-b77RbfmmF+Gjf[...]MSbp9PL4UUE9w==
npm notice total files:   3                                       
npm notice 
+ npm-cli-package@1.0.0
```


此时查看NPM官网中的个人账号信息，可以发现发布了该命令行工具的`1.0.0`版本。

如果需要发布[Scope包](https://docs.npmjs.com/creating-and-publishing-scoped-public-packages)，需要在NPM官网中创建一个组织（例如这里将ziyi222的账号作为组织，然后将账号ziyi222重命名成ziyi222222，并重新登录账号）：

``` javascript
AppledeMacBook-Pro:npm-cli-package ziyi2$ npm login
Username: ziyi222222
Password: 
Email: (this IS public) 673191402@qq.com
Logged in as ziyi222222 on https://registry.npmjs.org/.
```

重新修改`package.json`配置文件：

``` javascript
{
  "name": "@ziyi222/npm-cli-package",
  // ...
  "publishConfig": {
    "access": "public"
  }
}

```

使用`npm publish`发布：

``` javascript
AppledeMacBook-Pro:npm-cli-package ziyi2$ npm publish
npm notice 
npm notice 📦  @ziyi222/npm-cli-package@1.0.1
npm notice === Tarball Contents === 
npm notice 653B   package.json
npm notice 12.1kB README.md   
npm notice === Tarball Details === 
npm notice name:          @ziyi222/npm-cli-package                
npm notice version:       1.0.1                                   
npm notice package size:  5.1 kB                                  
npm notice unpacked size: 12.8 kB                                 
npm notice shasum:        fac560a42c43d276c9fee17c21887bedaf34bad1
npm notice integrity:     sha512-j/QScEgX+glfS[...]OxhhZcIavUhfg==
npm notice total files:   2                                       
npm notice 
+ @ziyi222/npm-cli-package@1.0.1
```



### 命令行工具下载安装和使用

开发者可以通过`npm install`  命令对命令行工具进行全局安装：

``` javascript
AppledeMacBook-Pro:npm-cli-package ziyi2$ npm install npm-cli-package -g
/usr/local/bin/npm-cli-package -> /usr/local/lib/node_modules/npm-cli-package/src/index.js
+ npm-cli-package@1.0.0
updated 1 package in 12.977s
```

由安装打印信息可以发现，最终命令行工具脚本链接指向了`/usr/local/bin/npm-cli-package`。安装成功之后，可以在项目中使用命令行工具指定的命令。


### 总结

命令行工具如果有功能需要扩展，可以再次通过`npm publish`命令发布新版本的NPM包，此时其他开发者可以通过`npm install`全局安装的形式进行工具升级，因此使用发布NPM包的形式可以使命令工具的版本升级和维护更方便。

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
