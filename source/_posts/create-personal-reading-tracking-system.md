---
title: 打造个人阅读追踪系统
date: 2018-01-13 21:03:34
categories: "生产力"
top: true
---

> 偶然看到[<<我也来打造一个个人阅读追踪系统>>](https://juejin.im/post/59d975b6f265da065f04d8ff)，觉得这个想法非常不错。于是利用工作外的时间，经过半个月的折腾，初步搭成了一个自动化的个人阅读追踪系统。在这个系统搭成后，才发现Read it Later = Read Never离自己好近。

## 工具

包括但不仅仅局限于下面的工具，这里我使用*Instapaper*作为稍后阅读的工具，*IFTTT*作为连接*Instapaper*和*Cloud*的中枢，而*WebTask*作为我的云端处理工具，负责真正请求各类API。

* [Github](https://github.com/)
* [Zenhub](https://www.zenhub.com/)
* [Instapaper](https://www.instapaper.com)
* [IFTTT](https://ifttt.com)
* [WebTask](https://webtask.io)

## 工作流程

在工作和生活中，我们经常会把文章保存到各种渠道中，比如印象笔记、Instapaper、Pocket等等，而这时，文章分散到各处，不利于我们了解和统计自己的阅读情况。这时是否可以在文章存档到各处后，自动在GItHub中创建一个Issues，而日后对文章的评论和圈点也会体现在Issues中，而且还有一个可视化的工具，详尽的展示了这一段时间内你的阅读情况，从而可以达到展示和督促的作用。

![工作流程](https://s1.ax1x.com/2018/01/14/pYyosf.jpg)

## 初始化WebTask

webTask作为*Serverless*架构的服务提供者，可以很方便的让我们在云端执行各类任务，在本文中，它的主要作用就是请求GitHub的API，以实现各类功能，比如给某个Issues添加评论、关闭某个Issues......

### 安装命令行工具

```js
npm install -g wt-cli
wt init <YOUR-EMAIL>
```

### 创建项目

新建一个node工程并添加index.js文件，补充如下代码：

```javascript
const Express = require('express')
const Webtask = require('webtask-tools')
const bodyParser = require('body-parser')
const app = Express()

app.use(bodyParser.urlencoded({ extended: false }))
app.use(bodyParser.json())

require('./routes/reading')(app)

module.exports = Webtask.fromExpress(app)
```

创建/routes/reading.js文件，这里以在仓库创建Issues为例，完整代码可参考[xuexiaoao/demo.serverless-mern](https://github.com/xuexiaoao/demo.serverless-mern)

```javascript
require('es6-promise').polyfill()
require('isomorphic-fetch')

const REPO_OWNER = 'xuexiaoao'
const REPO_NAME = 'reading'
const REPO_ID = 107835589

module.exports = (app) => {
  app.post('/reading', (req, res) => {
      const { GITHUB_ACCESS_TOKEN, ZENHUB_ACCESS_TOKEN, ZENHUB_ACCESS_TOKEN_V4 } = req.webtaskContext.secrets
      const { action, issue } = JSON.parse(req.body.payload)
      const { url, html_url, number } = issue

      console.info(`[BEGIN] issue updated with action: ${action}`)

      if (action === 'opened') {
        fetch(`${url}?access_token=${GITHUB_ACCESS_TOKEN}`, {
          method: 'PATCH',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ milestone: 1, }),
        }).then(
          () => console.info(`[END] set milestone successful! ${html_url}`),
          (e) => res.json(e),
        )
      } else if (action === 'milestoned') {
        fetch(`https://api.zenhub.io/p1/repositories/${REPO_ID}/issues/${number}/estimate?access_token=${ZENHUB_ACCESS_TOKEN}`, {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ estimate: 1 }),
        }).then(
          () => console.info(`[END] Set estimate successful! ${html_url}`),
          (e) => console.error(`[END] Failed to set estimate! ${html_url}`, e),
        )
        fetch(`https://api.zenhub.io/v4/reports/release/59ec29a2bedf0a4d94322b26/items/issues?access_token=${ZENHUB_ACCESS_TOKEN_V4}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
          body: `add_issues%5B0%5D%5Bissue_number%5D=${number}&add_issues%5B0%5D%5Brepo_id%5D=${REPO_ID}`,
        }).then(
          () => console.info(`[END] set release successful! ${html_url}`),
          (e) => console.error(`[END] Failed to set release! ${html_url}`, e),
        )
      }

      res.json({ message: 'issue updated!' })
    },
  )
}
```

这里有几个地方需要注意:

* REPO_OWNER：这是你的github名字
* REPO_NAME：这是你的仓库名称，之后所有的Issues都会创建到这里
* REPO_ID：这个id可以通过github api获得:https://api.github.com/users/${REPO_OWNER}
* GITHUB_ACCESS_TOKEN：这个没什么好说的就是在github上新建一个Personal access tokens
* ZENHUB_ACCESS_TOKEN：这个是在zenhub新建一个token
* ZENHUB_ACCESS_TOKEN_V4：这个官方api没有说明，但是可以打开zenhub对应仓库的浏览器开发者工具，任意找一个是 [api.zenhub.io](http://disq.us/url?url=http%3A%2F%2Fapi.zenhub.io%3AFORqzzRjWezjzHYTAiHvtcMGeAQ&cuid=3228622) 的请求，在 Request Headers中找到 x-authentication-token 对应的值就是这个 token 了

### 上传到webtask

这里需要使用上面准备的token，然后执行命令即可。由于使用webtask的上下文绑定工具*webtask-tools*，它会对一些敏感信息，比如我们的token进行加密，所以不必担心token泄漏的问题。

```shell
wt create index --bundle --secret GITHUB_ACCESS_TOKEN=$GITHUB_ACCESS_TOKEN --secret ZENHUB_ACCESS_TOKEN=$ZENHUB_ACCESS_TOKEN --secret ZENHUB_ACCESS_TOKEN_V4=$ZENHUB_ACCESS_TOKEN_V4
```

## 使用IFTTT自动化处理

那如何保证一有文章存入*Instapaper*中，就会新建一个Issues呢，这里就需要IFTTT这个神器了，这里不对IFTTT做详细介绍，如果有机会，可以另开一篇文章，专门介绍IFTTT。

得益于 IFTTT 非常丰富的第三方服务，IFTTT 可以直接创建 Instapaper 与 GitHub Issue 相集成的 Applet：[If new item saved, then create a new issue - IFTTT](https://ifttt.com/applets/54307045d-if-new-item-saved-then-create-a-new-issue)，就可以在当 Instapaper 新增文章的时候，自动在 GitHub 所指定的仓库 [xuexiaoao/reading](https://github.com/xuexiaoao/reading/issues) 中创建一个新的 Issue 并添加相应的标题、链接以及描述等相关信息

这里给出*把文章保存到Instapaper后，自动在仓库创建Issues的例子*，其他的操作都可以类似处理，如果有疑问，可以在本文留言。

<img src='https://s1.ax1x.com/2018/01/14/ptXYtJ.png' width="30%" height="30%" >

<img src='https://s1.ax1x.com/2018/01/14/ptONsf.png' width="30%" height="30%" >

## 集成ZenHub

现在仅仅会自动新建Issues，而我们还需要把Issues添加到Milestone中，就可以利用Zenhub的图表功能，可视化的查看和分析你的阅读情况。

Github提供WebHock功能，方便的让我们监控仓库的各类事件，从而触发某个自定义的功能，而这些功能我们已经放到了WebTask上面了。

![create_webhock](https://s1.ax1x.com/2018/01/14/ptjXPH.png)

这里你需要改的就是*Payload URL*，它是你在执行[上传到webtask](#上传到webtask)后得到的一个URL，其余的可以参考上图。



## 其他功能示例

### 在Instapaper做笔记

```javascript
  app.post('/reading-note', (req, res) => {
    const { GITHUB_ACCESS_TOKEN } = req.webtaskContext.secrets

    const title = req.query.title
    const note = req.body.note
    const highlightedText = req.body.highlightedText
    console.info('[BEGIN]', { title, note, highlightedText })

    let keyword = encodeURIComponent(title.replace(/\s/g, '+'))
    console.info('[KEYWORD]', keyword)

    fetch(`https://api.github.com/search/issues?q=${keyword}%20repo:xuexiaoao/reading%20is:open`)
      .then(response => response.json())
      .then(data => {
        console.info('[RESULT]', data)
        if (data.total_count > 0) {
          data.items.forEach(({ url, html_url }) =>
            fetch(`${url}/comments?access_token=${GITHUB_ACCESS_TOKEN}`, {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ body: `> ${highlightedText} <br/><br/> > ${note}` }),
            })
              .then(() => console.info(`[END] added comment successful! ${html_url}`))
              .catch(err => res.json('error', { error: err })))
          res.json({ message: 'Added comment into issue successful!' })
        }
        res.json({ error: 'Not Found!' })
      })
      .catch(err => res.json('error', { error: err }))
  })
```

### Instapaper归档文章

```javascript
  app.get('/reading', (req, res) => {
    const { GITHUB_ACCESS_TOKEN } = req.webtaskContext.secrets

    console.info('[BEGIN]', req.query)
    const title = req.query.title

    let keyword = encodeURIComponent(title.replace(/\s/g, '+'))
    console.info('[KEYWORD]', keyword)

    fetch(`https://api.github.com/search/issues?q=${keyword}%20repo:xuexiaoao/reading`)
      .then(response => response.json())
      .then(data => {
        console.info('[RESULT]', data)
        if (data.total_count > 0) {
          data.items.forEach(({ url, html_url }) =>
            fetch(`${url}?access_token=${GITHUB_ACCESS_TOKEN}`, {
              method: 'PATCH',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ state: 'closed' }),
            })
              .then(() => console.info(`[END] issue closed successful! ${html_url}`))
              .catch(err => res.json('error', { error: err })))
          res.json({ message: 'Closed issue successful!' })
        } else {
          console.info('[RESULT]', data)

          fetch(`https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/issues?access_token=${GITHUB_ACCESS_TOKEN}`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ title }),
          })
            .then(response => response.json())
            .then(({ url, html_url }) => {
              console.info(`[END] issue created successful! ${html_url}`)
              fetch(`${url}?access_token=${GITHUB_ACCESS_TOKEN}`, {
                method: 'PATCH',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ state: 'closed' }),
              })
                .then(() => console.info(`[END] issue closed successful! ${html_url}`))
                .catch(err => res.json('error', { error: err }))
            })
            .catch(err => res.json('error', { error: err }))
        }
        res.json({ error: 'Finished achieve reading item!' })
      })
      .catch(err => res.json('error', { error: err }))
  })
```

## 总结

至此，一个自动化的阅读追踪系统已经搭建完成，现在我们就可以查看每天乃至每周的阅读情况了，你只需要定期的查看阅读效果，就可以对自己的阅读计划进行调整和变更。

当然还有一些问题需要处理，比如对文章进行分类以使这套系统更有参考性，是否需要把在Instapaper中的笔记保存到自己的知识库中。

## 参考资料

* [Serverless 实战：打造个人阅读追踪系统](https://blog.jimmylv.info/2017-06-30-serverless-in-action-build-personal-reading-statistics-system/)
* [我也来打造一个个人阅读追踪系统](https://juejin.im/post/59d975b6f265da065f04d8ff)