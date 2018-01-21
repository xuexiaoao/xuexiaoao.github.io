---
title: 持续优化Hexo
date: 2018-01-15 22:50:30
categories: "Hexo"
---

本文主要记录Hexo的优化历程，以备后面翻阅。

## 添加emoji支持

由于Hexo的Markdown语法并不支持emoji表情，所以只能通过第三方插件来实现此功能

### 安装

```shell
npm i hexo-renderer-markdown-it --save
npm install markdown-it-emoji --save
```

### 配置

编辑站点文件

```yaml
markdown:
  plugins:
    - markdown-it-footnote
    - markdown-it-sup
    - markdown-it-sub
    - markdown-it-abbr
    - markdown-it-emoji
```

## 支持Gitter聊天室

### 新建gitter.swig文件

>  在~/*主题*/layout文件夹下创建，添加如下内容

```javascript
{% if theme.gitter.enable %}                                                                                                                                                              
 <script>                                                                                                                                                                                  
   ((window.gitter = {}).chat = {}).options = {                                                                                                                                            
    room: 'Newdee/newdee'                                                                                                                                                                 
  };                                                                                                                                                                                      
 </script>                                                                                                                                                                                 
 <script src="https://sidecar.gitter.im/dist/sidecar.v1.js" async defer></script>                                                                                                          
  {% endif %}
```

### 配置

* 设置样式

  > 在~/themes/next/source/css\_custom/custom.sty设置打开聊天室Button样式

  ```css
  .gitter-open-chat-button {                                                                                                                                                  
        right: 20px;                                                                                                                              
        padding: 10px;                                                                        
        background-color: #777;                                                                                          
  }                                                                                                                                                                                     
  @media (max-width: 600px) {                                                                                                                                                  
      .gitter-open-chat-button,                                                                                             
      .gitter-chat-embed {                                                                    
          display: none;                                                                    
      }                                                                                       
  }
  ```

* 添加Gitter到主题中

  >  添加到body div前面

  ```javascript
  {% include './gitter.swig' %}
  ```

* 在主题配置文件开启gitter

  ```yaml
  # Gitter
  gitter:
    enable: true
  ```

## 支持字数统计

> wordcount支持字数统计、阅读时长

### 安装插件

```shell
npm install hexo-wordcount --save
```

### 开启插件

> 在主题配置文件修改

```yaml
post_wordcount:
  item_text: true
  wordcount: true
  min2read: true
```

## 统计阅读次数

### 集成leancloud

#### 获取app_key和app_id

在leancloud注册帐号，之后新建一个应用，在此应用里可以找到app_key和app_id

![get app_key & app_id](https://s1.ax1x.com/2018/01/20/pRyK8H.png)

#### 新建Class

> 在新建的应用中创建一个Class，名称必须是Counter，其余默认即可

![create class](https://s1.ax1x.com/2018/01/20/pRyUPg.png)

#### 配置

把app_key和app_id填充到主题配置文件中如下位置

```yaml
leancloud_visitors:
  enable: true
  app_id: yourapp_id
  app_key: yourapp_key
```
## 添加版权声明

### 添加声明信息

> 在~\themes\next\layout\_macro\post.swig合适位置添加如下内容，我是放在<footer class="post-footer">标签黎明

```javascript
{# 版权声明节点 #}
     <div>    
      {% if not is_index %}
      <ul class="post-copyright">
         <li class="post-copyright-link">
          <strong>本文作者：</strong>
          <a href="/" title="欢迎访问 {{ theme.author }} 的个人博客">{{ theme.author }}</a>
        </li>

        <li class="post-copyright-link">
          <strong>本文标题：</strong>
          <a href="{{ url_for(post.permalink) }}" title="{{ post.title }}">{{ post.title }}</a>
        </li>

        <li class="post-copyright-link">
          <strong>本文链接：</strong>
          <a href="{{ url_for(post.permalink) }}" title="{{ post.title }}">{{ post.permalink }}</a>
        </li>

        <li class="post-copyright-date">
            <strong>发布时间：</strong>{{ post.date.format("YYYY年M月D日 - HH时MM分") }}
        </li>  

        <li class="post-copyright-license">
          <strong>版权声明： </strong>
          本文由采用 <a href="https://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh" rel="license" target="_blank">保留署名-非商业性使用-禁止演绎 4.0-国际许可协议</a> </br>转载请保留以上声明信息！
        </li>
      </ul>
    {% endif %}
  </div>
```

### 设置声明样式

> 在~/themes/next/source/css\_custom/custom.styl添加样式

```css
.post-copyright {
    margin: 2em 0 0;
    padding: 0.5em 1em;
    border-left: 3px solid #FF1700;
    background-color: #F9F9F9;
    list-style: none;
}
```

## 显示文章更新日期

> 因为我用的是Next主题，所以操作非常简单，只需修改主题配置文件即可

```yaml
post_meta:
  item_text: true
  created_at: true
  updated_at: true
  categories: true
```

## 支持文章置顶

### 下载插件

```shell
npm uninstall hexo-generator-index --save
npm install hexo-generator-index-pin-top --save
```

### 添加置顶标签

> 打开~/themes/next/layout/_macro/post.swig，在\<div class="post-meta">添加如下代码

```javascript
{% if post.top %}
  <span class="post-meta-item-icon">
  	<i class="fa fa-thumb-tack"></i>
  </span>
  <font color=7D26CD>置顶</font>
  <span class="post-meta-divider">|</span>
{% endif %}
```

### 添加标记

> 在需要置顶的文章的 `Front-matter` 中加上 `top: true` 即可，

比如：

```yaml
title: 打造个人阅读追踪系统
date: 2018-01-13 21:03:34
categories: "生产力"
top: true
```

## 支持文章分享

### 修改过气API

> 这是目前找到的唯一支持https的分享插件，但是分享到微信的api有改动，所以需要修改~\themes\next\source\lib\needsharebutton\needsharebutton.js如下位置

```javascript
/**
找到imaSrc定义的位置，改为如下
**/
var imgSrc = "https://pan.baidu.com/share/qrcode?w=400&h=400&url=" + encodeURIComponent(myoptions.url);
```

### 启用分享

> 因为next主题已经集成needsharebutton插件，所以只需要开启即可

```yaml
needmoreshare2:
  enable: true
  postbottom:
    enable: true
    options:
      iconStyle: box
      boxForm: horizontal
      position: bottomCenter
      networks: Weibo,Wechat,Douban,QQZone,Twitter,Facebook
  float:
    enable: true
    options:
      iconStyle: box
      boxForm: horizontal
      position: middleRight
      networks: Weibo,Wechat,Douban,QQZone,Twitter,Facebook
```

