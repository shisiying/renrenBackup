# renrenBackup

A backup tool for renren.com


# 人人网信息备份工具

## 基本配置

依赖 Python 3.6.5+（其他版本我没测试过，因为用了 f-string，所以应该是要这个版本起跳）

用 pipenv 构建虚拟环境

```bash
pipenv --python 3.6.5
pipenv install
pipenv shell
```

或者用 virtualenv 构建

```bash
virtualenv env
source env/bin/activate
pip install -r requirements.txt
```

## 抓取

直接运行 `fetch.py` 即可，相关参数见下，不输入用户名密码是不会抓取的，不带各种抓取参数就是只登陆不抓取

```bash
$ python fetch.py --help
usage: fetch.py [-h] [-s] [-g] [-a] [-b] email password

fetch renren data to backup

positional arguments:
  email               your renren email for login
  password            your renren password for login

optional arguments:
  -h, --help          show this help message and exit
  -s, --fetch-status  fetch status or not
  -g, --fetch-gossip  fetch gossip or not
  -a, --fetch-album   fetch album or not
  -b, --fetch-blog    fetch blog or not

$ python fetch.py email@renren.com passwordAtRenren -s -g -a -b
```


## 展示

直接运行如下命令，即可在本机浏览器打开 `localhost:5000` 看到展示

```bash
python web.py
```


## Log

#### 2018-07-31

把抓取的入口优化了下，用户名密码在抓的时候再输，免得要去改 config 还可能不小心提交到 git 上去


#### 2018-07-30

不死心，还是想搞定登录，其实这个应该就是用某个第三方库做了下 AES 之类的加密，爬一下请求过程吧

把 http://s.xnimg.cn/a89037/nx/apps/login/login.js 给拿下来并做格式化并分析

1. 第一步是在 865 行，DOM ready 后调 554 行的 `Window.getKeys`，向 "key.jsp?generateKeypair=true" 发一个请求申请服务端生成 key
2. 然后调 389 行的 `cryption.getKeys`，向 http://login.renren.com/ajax/getEncryptKey 发请求获得一个 json
3. 把里面的 `rkey` 保留下来，回头真正登录时要带上这个参数。用里面的 `e`, `n`, `maxdigits` 几个参数生成一个 rsa_key `T`。基本可以参考 http://aiddroid.com/javascript-rsa-encryption/ 里的代码 797 行，对应的 `.e` 是里面的 `factor`，`.n` 是里面的 `key`
4. 最终触发登录操作的是 560 行的 `onsubmit`，里面的 u 对应的是前面拿到的 rkey，所以必然会执行后面的两个 encrypt，对应的调用时 410 行，把输入的 `e` 跟之前生成的 rsa_key `T` 一起去加密，分别是把邮箱和密码加密，然而最后的提交 507 行的 `s()` 里只用了 `c`，也就是密码加密后的串

看了一下就是用的上面 aiddroid 里的那个算法，把加密部分看懂了后，密码不会超长，所以直接就是把密码按字节从左到右映射到大数 x 上，然后做 x**e % n 再转成 16 进制就是要的加密密码了

另外吐槽下登录的 URL，参数里带了个时间戳，但是这个戳不是用的实际任何一种语言的 timestamp，而是自己构建了一套，并且中间有些字段是缺失的（比如没有 `day_of_month` 但有 `day_of_week`，也没有分钟数），也不保证等长（上午 9 点就是 9，晚上九点就是 21），还好现在 Py3.6+ 有模板字符串可以直接填


#### 2018-07-28

本来还想把分享抓下来，不过看了下请求，分享页面只有 HTTP 拿回来的 HTML，并没有 Json 好拿，而且格式太多，有 日志、视频、相册、照片、链接 等，解析和展示都不方便。另外分享的本地评论不多，但全站评论抓起来规模就非常可观，特别是某些热门的内容（如果跳过抓头像，应该也还好？）

还是先跳过这个，回头有空再说，把已有页面结构梳理好，先做一版纯 HTML 的，方便输出


#### 2018-07-27

把日志也都抓了下来。日志列表在刚进入的时候看不到 ajax 请求，一翻页就能拿到 ajax 请求，这下直接摸数据就好

日志详情页不是 ajax 请求来的，也不知道怎么构建，看了下 HTML 格式，直接裸抓了。人人吐出来的 HTML 日志内容部分在一行，那就摸那一行，连正则都不用，直接 find 就可以搞定，只是有个坑的地方是人人 `\r` 和 `\n` 乱用，一开始多抓了一部分无效内容

日志的评论还是 ajax 请求来的，但是没看到点赞名单的请求，不想裸解析 HTML，按原来的 URL 和格式猜了个点赞请求，直接能拿到，完美

把展示时的翻页逻辑整合成了一个 Jinja 的 macro，也不用到处复制粘贴重复代码了

考虑了下未来的展示，为了方便查看，还是尽可能的用 Jinja 吐，这样直接保存 HTML，回头也不用启 flask 也可以离线看。当然，从学习和炫技的角度出发，还是可以搞前后端分离，也去学下 Vue.js


#### 2018-07-25

抓图的时候偶尔可能会 ConnectionError，果断把超时和重试机制给加上

人人的评论还分当前评论和全站评论，很多好玩的回复都在全站评论里，还是考虑抓个全站评论


#### 2018-07-24

抓相册时没找到拿相册列表的 ajax 请求，只能去裸解析页面，不过在页面内嵌的 js 里有相关数据，用不上 `requests-html` 这么重的东西，直接正则表达式取出来就行

本来打算是按 相册列表 -> 相册 -> 照片列表 -> 照片 的顺序去拿，后面发现其实只要拿到某个相册的任意一张照片，就可以在 ajax 请求里拿到相册信息和里面的所有照片信息，倒是省事了不少

抓到本地来图全挂，不允许跨域调，下一步还是考虑怎么把图片抓到本地来并做映射

目前会挂的图有这么几个来源

1. 留言板的头像
2. 留言板的附件
3. 评论和点赞的用户头像（其实跟 1 差不多，不过 1 里面是留言相关的，不是用户最新头像）
4. 相册封面（其实是里面的某张照片）
5. 照片（有多个尺寸，我取最大的了，在本地压缩尺寸）

域名也存在好几个，干脆都存到本地，并保持目录结构，映射过来就是 `http://xx.xxx.cn/aa/bb/cc/dd.gif` -> `/static/img/xx_xxx_cn/aa/bb/cc/dd.gif`，抓的时候去一下重，节省网络开销，然后把本地的展示直接替换就可以了

表情和礼物什么的看了下还不会被跨域封，替换起来比较麻烦，先不处理


#### 2018-07-23

本来想弄相册，看了下相册列表是在相册列表页的 HTML 里直接返回的，不是 json，所以要解析一下页面，引入 `requests-html` 这个库

后面发现还有留言板，结构更简单，打算先下手这个

翻页的时候发现留言板其实也有 ajax 请求，可以不用解析页面，这样就更简单了，直接拿。有几个需要注意的地方

1. 留言板里每条留言的头像都是用户在当时状态的头像，所以不好塞 User 表里，而是直接记录在留言里方便点
2. 时间给的字符串不是时间戳，自己转一下
3. 留言内容在好几个地方有，filterOriginBody 是最原始的，但是表情没转义，想拿转义后的在 filterdBody 或者 body 里，但是又加了很多没必要的东西，需要滤一下

处理展示的时候发现跨域的图片拿的有问题，回头还是得把有跨域限制的图片爬到本地来（包括留言板的头像和附图等）


#### 2018-07-22

数据爬下来了怎么存也是个问题，打算用 sqlite，因为数据量也不大，但是裸操作 sqlite 还是太麻烦，搜了下 ORM，找到 peewee 这个库，研究了下怎么用

考虑爬取后的展示问题，打算用 flask 做服务，Vue.js 做前端，semantic-ui 做样式

把状态的获取都搞定了，包括

1. 所有状态信息。包括 点赞、分享、评论 数
2. 所有状态的前 8 个点赞人（人人只返回了前 8 个）
3. 所有状态的所有评论

并用 flask 做了展示，先没上 Vue.js，只是裸写了一顿，熟悉了下 semantic-ui 的逻辑


#### 2018-07-21

牙龈发炎折腾了快一周，拖稿了

研究了半天现在登录的加密，混淆过的 js 读的脑仁疼，先跑通再说，直接从 Chrome 的请求里把加密后的 `password` 和 `rkey` 拿出来，就可以登录得到可用的 session 了

第一步先拿状态列表

状态直接是通过 AJAX 请求拿回来一段 json，这个就方便很多了，看了下返回的数据结构，有用的大概如下（部分数据已模糊化）

```json
{
    "count": 999,           // 总共有多少条状态
    "doingArray": [         // 当前页的状态列表
        {
            "comment_count": 20,            // 评论数
            "content": "要的留邮箱",        // 内容
            "createTime": "1416289808000",  // 时间戳，后面还有个 dtime 用来显示，我们爬就不用了
            "id": 12345,                    // 编号，后续拿评论什么的都用的上
            "repeatCountTotal": 2,          // 转发数，还有个 repeatCount 不知道干嘛的（直接转发？）
            "rootContent": "谁有？",        // 转发原文（如果是转发，否则没这个和下面几个字段）
            "rootDoingUserId": 23456,       // 转发原用户 id
            "rootDoingUserName": "张三",    // 转发原用户名
        },
    ]
    "likeInfoMap": {        // 点赞信息，还有个 likeMap 不知道干嘛的，跟页面对比是这个有用
        "status_12345": 1,                  // 格式是 status_id，用上面的来找就行了
    }
}
```

每页 20 条，页码从 0 开始，所以最大页码就是 `Math.ceil(count/20) - 1`，一路翻过去就好

重新爬的时候发现调登录可能会遇上要输入验证码，不如直接从 Chrome 里把 cookie 拿出来更方便

#### 2018-07-12

试图直接从 Chrome 里拷 Cookie 出来用，没用成功，还是走一遍 login 好了。之前找的那几个都是用明文登录的方式，似乎现在都不行了，抓了下请求包，还没完全搞明白官方是怎么加密的，但是在浏览器里把 ajaxLogin 那个请求的 form data 扒出来，直接提交后就可以获取到正确的 requests.session

#### 2018-07-11

想起来自己的人人账号也好久没用了，说不定这个站哪天都不在了，等官方有导出工具还是别想了，自己来吧

找了一圈找到了如下几个备份工具，不过看说明从 2018 年某刻后都不能用了，自己学着改改看

* https://github.com/xinyu3ru/renren-bak
* https://github.com/threegirl2014/RenRenDownload
* https://github.com/kngxscn/backup-renren-status
