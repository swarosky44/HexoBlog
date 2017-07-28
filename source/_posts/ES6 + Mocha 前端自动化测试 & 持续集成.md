---
title: ES6 + Mocha 前端自动化测试 & 持续集成
date: 2017-07-28 18：00
tags:
  - TDD
  - JavaScript
categories: TDD
---

就目前两年左右的工作经验来说，任职过的公司很少有强制的单元测试要求，大多数时间都是被需求和业务追着屁股跑 🙃
不过写单元测试还是蛮爽的，见下图：
![](http://7xro5v.com1.z0.glb.clouddn.com/tdd.jpeg)
怎么样，是不是看着很爽 🤓

<!-- more -->

## 配置流程
这次主要是配置 Webpack + ES6 + Karma + Mocha + Chai + Travis-ci + Coveralls;
1. 写好需要单元测试的代码；
2. 正常配置好 `webpack.config.js`，用于 babel 编译和 webpack 代码打包；
3. 安装 karma 测试过程管理工具，并配置好 `karma.conf.js`;
4. 安装 Mocha + Chai 作为测试框架和断言库，修改 `karma.conf.js`;
5. 配置 `.travis.yml` 文件
6. 配置 `.coveralls.yml` 文件

## 实现
### 业务代码
这里就随便写一个单链表用于测试
```JavaScript
class Node {
  constructor(value) {
    this.value = value;
    this.next = null;
  }

  setNextNode(node) {
    this.next = node;
  }

  getNextNode() {
    return this.next;
  }
};

class LinkedList {
  constructor() {
    this.head = null;
    this.length = 0;
  }

  append(value) {
    // TODO
  }

  insert(position, value) {
    // TODO
  }

  removeAt(position) {
    // TODO
  }

  valueOf() {
    // TODO
  }
}

export default LinkedList;
```
因为之后会配置 webpack + babel 所以可以放心的使用 ES6 的语法。

### 配置 webpack
首先装好依赖
```bash
npm i -s webpack babel-core babel-loader babel-preset-es2015 istanbul-instrumenter-loader
```

然后写好 `webpack.config.js` 的配置文件

```JavaScript
const path = require('path');
module.exports = {
  devtool: 'inline-source-map',
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: ['babel-loader'],
      },
      {
        test: /\.js$/,
        exclude: /node_modules|\.spec\.js$/,
        use: {
          loader: 'istanbul-instrumenter-loader',
          options: {
            esModules: true,
          },
        },
        enforce: 'pre',
      },
    ],
  },
  resolve: {
    extensions: ['.js'],
    alias: {
      'src': path.resolve(__dirname, './src'),
    },
  },
};
```
现在都是 webpack2 的配置了，相关语法不熟悉的话，最好是去看一下[webpack2 文档](https://doc.webpack-china.org/configuration/)
`rules` 就是以前的 loaders：
  - test：文件检测规则；
  - exclude：排除在外的文件目录；
  - use：所使用的 loaders，可以配置成如下的形式：
  ```JavaScript
  use: [
    {
      loader: 'babel-loader',
      options: {
        presets: ['es2015']
      }
    }
  ]
  ```
  这样就可以不要配置 .babelrc 文件了, 可以自由选择配置方式 ~

  ```JSON
  {
    "presets": ["es2015"]
  }
  ```

  ### 配置 Karma
  Karma 是一个测试过程管理工具，相当于一个整个配置过程的一个收敛点。
  首先安装所需依赖
  ```bash
  npm i -s karma karma-chai karma-mocha karma-mocha-reporter karma-coverage-istanbul-reporter karma-phantomjs-launcher karma-sourcemap-loader karma-webpack mocha mocha-lcov-reporter chai
  ```

  然后编写配置文件 `karma.conf.js`

  ```JavaScript
const webpackConfig = require('./webpack.config.js');
const path = require('path');

module.exports = function(config) {
  config.set({
    frameworks: ['mocha', 'chai'],
    files: [
      'test/*.spec.js',
    ],
    preprocessors: {
      'test/*.spec.js': ['webpack', 'sourcemap'],
    },
    reporters: ['mocha', 'coverage-istanbul'],
    webpackServer: {
      noInfo: true,
    },
    plugins: [
      'karma-coverage',
      'karma-webpack',
      'karma-mocha',
      'karma-chai',
      'karma-phantomjs-launcher',
      'karma-sourcemap-loader',
      'karma-mocha-reporter',
      'karma-coverage-istanbul-reporter',
    ],
    webpack: webpackConfig,
    port: 9876,
    colors: true,
    autoWatch: false,
    browsers: ['PhantomJS'],
    singleRun: true,
    coverageIstanbulReporter: {
      reports: ['html', 'lcovonly', 'text-summary'],
      dir: path.resolve(__dirname, 'coverage'),
      fixWebpackSourcePath: true,
      'report-config': {
        html: {
          subdir: 'html',
        },
      },
    },
  });
};
```
讲解一下各个参数的含义：
- frameworks：所用的测试框架；
- files: 需要加载到测试浏览器的文件；
- preprocessors: 加载之前需要预处理的文件；
- reporters：输出测试结果的工具；
- webpackServer： webpack 在 karma 中的配置，我这里配置了在测试过程中不要输出 webpack 打包过程的信息；
- plugins：所用到的插件；
- webpack：webpack 的配置；
- port：测试服的端口号；
- colors：报告中的颜色是否开启；
- autoWatch: 是否观察文件自动变化；
- browsers：测试浏览器；
- singleRun：是否只运行一次；
- coverageIstanbulReporter：输出的报告形式。

从 webpack 的配置中开始，就可以看到一些 istanbul 字样的配置信息，这些用于打点的工具，主要服务于测试覆盖率。如果不需要用 webpack + ES6 的话，测试覆盖率可以直接用 `karma-coverage` 工具。
由于 webpack 会将代码打包，并且加入大量的 polyfills，导致源码的体量增大，所以覆盖率会测试不准。
这里用 `istanbul-instrumenter-loader` 可以在打包之前就完成打点的工作。
理论上，到这一步如果配置完成了，并且依赖都成功安装了。就可以开始单元测试了。

### 测试代码
```JavaScript
import LinkedList from 'src/linkedList';
import { expect } from 'chai';

describe('Test LinkedList append function =>', () => {
  const ll = new LinkedList();
  ll.append(1);
  ll.append(2);

  it('length should be 1', () => {
    expect(ll.length).to.be.equal(2);
  });

  it('the value of the head should be 1', () => {
    expect(ll.head.value).to.be.equal(1);
  });
});

describe('Test LinkedList valueOf function =>', () => {
  const ll = new LinkedList();
  ll.append(1);
  ll.append(2);

  it('the result should equal [1, 2]', () => {
    expect(ll.valueOf()).to.deep.equal([1, 2]);
  });
});

describe('Test LinkedList insert function =>', () => {
  const ll = new LinkedList();
  ll.append(1);
  ll.append(2);
  ll.insert(1, 3);

  it('the result should equal [1, 3, 2]', () => {
    expect(ll.valueOf()).to.deep.equal([1, 3, 2]);
  });
})

describe('Test LinkedList removeAt function =>', () => {
  const ll = new LinkedList();
  ll.append(1);
  ll.append(2);
  ll.append(3);
  ll.removeAt(1);

  it('the result should equal [1, 3]', () => {
    expect(ll.valueOf()).to.deep.equal([1, 3]);
  });
})
```
具体的 [Mocha](https://mochajs.org/) + [Chai](http://chaijs.com/) 的语法可以自己去官网学习一下，非常简单的。

### Travis 集成
Travis 是一个持续集成的工具，当我们成功配置好之后，可以让我们一提交代码到 github 仓库之后，就立即执行我们的单元测试。
然后我们还可以简单配置一下 github 项目的 Readme.md，获得一个小 badage。

首先去 [travis-ci](https://travis-ci.org/) 的官网上完成注册（用 github 账号登录）。然后在项目列表：
![](http://7xro5v.com1.z0.glb.clouddn.com/travis.jpeg)
选中自己要开启持续集成功能的项目，并在该项目里面添加一个 .travis.yml 文件。

```yml
language: node_js
sudo: true
node_js:
  - stable
cache:
  directories:
    - node_modules
before_install:
  - npm i
script:
  - npm run test
after_success:
  - npm run coverage
```

这份配置的各项参数：
- language: 告诉服务器我们所使用的语言，我们用的是 nodejs；
- sudo: 是否开启管理员权限；
- node_js: node 的版本，我们选择稳定版；
- cache：每次重新执行测试，都需要重装环境和依赖，这里可以选择一些东西进行缓存，我们就缓存了我们的依赖库 node_modules；
- before_install: 依赖安装前需要做的事情；
- script：测试的执行命令；
- after_success: 集成测试覆盖率；

然后 push github，可以到 travis 上面看到如下：
![](http://7xro5v.com1.z0.glb.clouddn.com/travis1.jpeg)
![](http://7xro5v.com1.z0.glb.clouddn.com/travis2.jpeg)

点击小猫咪头像边上的 badage，然后选择 markdown 格式，复制下面的代码，添加到 readme.md 文件中，就可以获得一次装逼的机会。🤓
![](http://7xro5v.com1.z0.glb.clouddn.com/travis3.jpeg)

BTW：这里注意要将 `karma.conf.js` 文件中的配置 - `authoWatch： false` & `singleRun: true`。否则会一直卡住 travis 服务器。

### coveralls集成
这个是用于集成测试覆盖率的，也比较简单。先去[coveralls](https://coveralls.io/)完成注册（github 账号登录），同样去列表选中自己要开通的项目。然后安装一个依赖：
```
npm i -s coveralls
```
这是用于向 coveralls 上传单元测试完成后的 lcov.info 文件的。
在项目里添加一个文件 .coveralls.yml

```yml
service_name: travis-ci
repo_token: xxx
```

service_name 是写死的，不需要理会。
repo_token：是开启项目之后，detail 页面给你的 repo_token，作为账户的唯一识别。

然后在 `package.json` 文件下添加一条命令：
```JSON
"coverage": "cat ./coverage/lcov.info | ./node_modules/coveralls/bin/coveralls.js"
```

这里集成的整个逻辑其实不复杂：
 - 首先我们完成了单元测试的代码
 - git push 代码之后 travis 会收到然后执行我们单元测试
 - 单元测试完成后（即.travis.yml 中配置的 after_success 触发钩子）会执行 `npm run coverage`，向 coveralls 上传我们的覆盖率报告。
 - coveralls 会收到请求，然后生成相关数据。

同样我们也可以收获一个装逼用 badage ~

BTW: 这里我其实遇见了一个挺麻烦的问题，就是在 travis 的服务器上执行 `npm run coverage` 后，并没有成功执行，而是抛出了个错误:
```
Bad response: 422 {"message":"Couldn't find a repository matching this job.","error":true}
```
具体我也不清楚要怎么解决，貌似 coveralls issue 蛮多的，网上也搜了很多，但是都没用。
最后我关闭了项目（列表勾选那里），然后重新开启，更换了 repo_token 就行了。

## 总结
整个过程不算复杂，就是很繁琐。耐心点跟着配置就好了，遇见问题多 google 总能解决的。
![](http://7xro5v.com1.z0.glb.clouddn.com/travis4.jpeg)

> 参阅的文章太多啦~ 文档也蛮多的~
> [项目地址](https://github.com/swarosky44/Unit-Test-CI)
