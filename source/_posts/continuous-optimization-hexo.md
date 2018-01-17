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
npm i hexo-filter-github-emojis --save
```

### 配置

编辑站点文件

```yaml
vim _config.xml
githubEmojis:
  enable: true
  className: github-emoji
  unicode: false
  styles:
  localEmojis:
```

### 兼容Hexo

默认使用会在页面显示一个a标签而不是emogi表情，所以需要修改一些地方。

* 修改*themes/next/layout/_partials/head.swig*

  ```javascript
  <script type="text/javascript" id="hexo.configurations">
    var NexT = window.NexT || {};
    var CONFIG = {
      root: '{{ theme.root }}',
      scheme: '{{ theme.scheme }}',
      sidebar: {{ theme.sidebar | json_encode }},
      fancybox: {{ theme.fancybox }},
      tabs: {{ theme.tabs.enable }},
      motion: {{ theme.use_motion }},
      duoshuo: {
        userId: '{{ theme.duoshuo_info.user_id | default() }}',
        author: '{{ theme.duoshuo_info.admin_nickname | default(__('author'))}}'
      },
      algolia: {
        applicationID: '{{ theme.algolia.applicationID }}',
        apiKey: '{{ theme.algolia.apiKey }}',
        indexName: '{{ theme.algolia.indexName }}',
        hits: {{ theme.algolia_search.hits | json_encode }},
        labels: {{ theme.algolia_search.labels | json_encode }}
      },
      {# ↓ 在这里修改，其他未动 ↓ #}
      {# 添加 emojis 参数 #}
      emojis: {
        className: '{{ config.githubEmojis.className | default('github-emoji') }}'
      }
    };
  </script>
  ```

* 修改*themes/next/source/js/src/utils.js*

  ```javascript
  // 注意看 CONFIG.emojis.className 那一行，其他未动
  $('.content img')
    .not('[hidden]')
    .not('.group-picture img, .post-gallery img, img.' + CONFIG.emojis.className)
    .each(function () {
      var $image = $(this);
      var imageTitle = $image.attr('title');
      var $imageWrapLink = $image.parent('a');

      if ($imageWrapLink.size() < 1) {
          var imageLink = ($image.attr('data-original')) ? this.getAttribute('data-original') : this.getAttribute('src');
        $imageWrapLink = $image.wrap('<a href="' + imageLink + '"></a>').parent('a');
      }

      $imageWrapLink.addClass('fancybox fancybox.image');
      $imageWrapLink.attr('rel', 'group');

      if (imageTitle) {
        $imageWrapLink.append('<p class="image-caption">' + imageTitle + '</p>');

        //make sure img title tag will show correctly in fancybox
        $imageWrapLink.attr('title', imageTitle);
      }
    });

  $('.fancybox').fancybox({
    helpers: {
      overlay: {
        locked: false
      }
    }
  });
  ```

  ​