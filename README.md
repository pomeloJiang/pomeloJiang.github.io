# Hexo博客搭建

###  1、安装hexo

```
wget -qO- https://raw.github.com/creationix/nvm/master/install.sh | sh
source ~/.profile
nvm install stable
npm install -g cnpm --registry=https://registry.npm.taobao.org
npm install hexo --save
```

### 2、安装插件
```
npm install hexo-generator-index --save #索引生成器
npm install hexo-generator-archive --save #归档生成器
npm install hexo-generator-category --save #分类生成器
npm install hexo-generator-tag --save #标签生成器
npm install hexo-server --save #本地服务
npm install hexo-deployer-git --save #hexo通过git发布（必装）
npm install hexo-renderer-marked@0.2.7--save #渲染器
npm install hexo-renderer-stylus@0.3.0 --save #渲染器
```

### 3、初始化本地博客文件夹
```
mkdir blog && cd blog
hexo init
hexo 
```

### 4、将文章源码复制到blog文件夹下替换

### 5、HEXO命令
```
hexo generate
hexo server
hexo server -p 2333 //指定端口
hexo deploy
hexo help
```

### 6、主题
Concise:https://github.com/sanonz/hexo-theme-concise
编译器 hexo-renderer-stylus切换为hexo-renderer-less

```bash
npm install hexo-renderer-less --save
```
安装
```bash
git clone https://github.com/sanonz/hexo-theme-concise.git themes/concise
```