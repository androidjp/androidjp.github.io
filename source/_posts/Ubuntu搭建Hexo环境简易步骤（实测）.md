---
title: Ubuntu搭建Hexo环境简易步骤（实测）
date: 2016-06-25 11:27:50
categories:
- Linux
- Hexo
tags:
- Hexo
- Ubuntu

---
## Ubuntu环境
Ubuntu14.04LTS

---------------
## 环境搭建

### 搭建Nodejs环境
1. 解压4.4.4版本的Node.js
<br/>（[中文网](http://nodejs.cn/)或者[官网](https://nodejs.org/en/)下载压缩包直接右键解压
<br/>或者：
<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;①tar.gz包： 命令行：
        tar -zxvf node-xxxx.tar.gz
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;②tar.xz包：命令行：
        xz -d node-xxxx.tar.xz
          tar xvf node-xxxx.tar
2. 设置node.js的环境变量：
<br/>
  1) sudo gedit /etc/profile
  <br/>2) 编辑profile文件，加入下列命令：
        #set nodejs environment
          export HOME=/home/androidjp
          export NODE_HOME=$HOME/Nodejs/node-v4.4.4-linux-x64
          export PATH=$NODE_HOME/bin:$PATH
          export NODE_PATH=$NODE_HOME/lib/node_modules
  3) 保存，退出
  <br/>4) 命令行执行：source /etc/profile<br/>
3. 安装npm
推荐这样：<br/>
        sudo apt-get install apmd
<br/>也可以：<br/>
sudo apt-get install curl<br/>
curl http://npmjs.org/install.sh | sudo sh


### 安装Git
    一般方式：
    sudo apt-get update
    sudo apt-get install git    
    git --version 或 git -v


### 安装hexo
1. cd 到你要本地写博客的根目录，比如在用户主目录创建一个目录为‘blog’：
        mkdir blog
2. cd到blog这个目录
        cd blog
3. 在这个目录中安装复制生成 hexo 的配置信息
        sudo npm install -g hexo  或者 npm install-g hexo
4. 初始化 hexo
        hexo init
5. 生成静态页面
        hexo g    或者    hexo generate
5. 本地开启服务
        hexo server
6. 浏览器输入 http://localhost:4000 测试是否出现hexo页面
7. github 创建一个新的仓库（repository），命名为：
        your_user_name.github.io
  这里，选择这个新仓库的地址，可以用https ，也可以用 ssh。
8. 返回到本地你的blog目录中
9. blog本目录下命令行打开_config.yml进行配置
        gedit \_config.yml  或   vi \_config.yml
   在文件的最后的 ‘deploy’ 处，添加如下命令（注意：每个冒号（：）后面跟一个空格，不然会失效！）
          deploy:
          type: git
          repo: https://github.com/yourUserName/your_user_name.github.io.git
          branch: master
10. 保存配置        
          npm install hexo-deployer-git --save
11. 部署到github
          hexo deploy
12. 完成！此时可以输入 http://your_user_name.github.io/  看是否成功部署～

### hexo每次部署步骤（注意cd到blog根目录）
    hexo clean
    hexo g
    hexo deploy

### hexo常用命令
    hexo new"postName" #新建文章
    hexo new page"pageName" #新建页面
    hexo generate #生成静态页面至public目录
    hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
    hexo deploy #将.deploy目录部署到GitHub
    hexo help # 查看帮助
    hexo version #查看Hexo的版本

> 额外可能用到的linux命令：
    sudo apt-get remove softwareName   卸载指定的软件


### 使用Atom编辑器编辑md文件
关于 markdown 的各种语法，可参考文章：[markdown学习笔记](http://www.jianshu.com/p/4Q3aay)
1. 安装编辑器（推荐命令行下载安装）
--> 可选择直接下载安装[编辑器下载地址](https://atom.io/)
        sudo add-apt-repository ppa:webupd8team/atom  
        sudo apt-get update  
        sudo apt-get install atom
2. 打开项目(File  -->  Open Folder --> 选择 \_posts 目录导入)
3. 安装插件
  一般插件安装方式：

  方式一：通过atom菜单的setting直接搜索project进行install（本人做法：'ctrl+shift+p'，输入框输入“welcome” --> Welcome Guide --> Install a project --> 输入你要找的插件名 --> 点击安装）
  <br/>方式二：上 Github 下载插件,放在 atom 目录下的 package,重开打开软件会自动识别
  <br/>方式三：使用 apm 命令安装,例:apm install run-in-browser

4. 图片上传辅助
  利用[七牛云](http://yotuku.cn/)等上传图片，获取链接
  或者参考![此网站](http://www.tuicool.com/articles/umEBVfI),辅助图片上传

  例如：安装rib网页预览插件
  <br/>![pic_install_rib](http://i4.piimg.com/567571/ad8a58ed6f05aacc.png)

### 相关问题
1. github绑定自己的域名（这里示例为：腾讯云域名，绑定github.io）
<br/>  1.1 首先，在你的github.io 仓库根目录，新建一个文件‘CNAME’，里面填入你的域名
<br/>![](http://i1.piimg.com/567571/2ed1bbbca456965a.png)
<br/>
<br/>![](http://i1.piimg.com/567571/ddc8c57535459f1d.png)
<br/>  1.2 接着，在你的域名平台上，加入两个解析（解析github IP）
<br/>![](http://i1.piimg.com/567571/569aef773709826f.png)


----
### 参考文章
http://blog.fens.me/hexo-blog-github/<br/>
http://www.jianshu.com/p/465830080ea9
