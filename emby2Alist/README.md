---
title: emby/jellyfin挂载alist转直链
date: 2021/09/06 22:00:00
---

### 文章更新记录 

2024/04/27

1.升级路由缓存配置,缓存的key值可自定义表达式

2.升级路由缓存配置,加入二级缓存开关,注意此仅针对直链,添加阶段为进入单集详情页,cilentSelfAlistRule中的和首页直接播放的不生效,
自行根据实际情况使用,只在缓存中没有直链且进入详情页才会查alist预热缓存,15分钟内第二次进入相同详情页命中缓存不再查询alist,
忽略直链通知的文本内容,只预热了缓存并没有跳转,再次点击播放就会直接读取预热后的直链缓存

2024/04/25

1.媒体项目过滤添加可能为第三方使用的海报推荐接口

2024/04/22

1.修复[emby > 4.8.3.0]直链通知的窗口持续时间

2.更改易混淆的变量名,修复多版本媒体的直链缓存key

3.提供参数控制302的直链文件是否保留码率选择,不保留客户端将无法手动切换至转码

4.提供转码负载失败的降级措施,默认为"redirect"的302,进入转码接口后官方客户端会将播放环境改为HLS,
如果内部重定向到直链是不生效的,客户端会死循环,处理方案为先响应了一次403报错,依赖客户端默认的3次重试机会,
第一次重试就会重新初始化为普通播放环境播放302直链内容,
"proxy": 这个单纯只是作为兜底方案,转给原始媒体服务处理,如果emby客户端自己带有转码参数,原始服务也会走转码,
客户端可自行选择最大码率来切换回直链的302

2024/04/21

1.还原以兼容115部分客户端拖动进度条bug

2.对直链通知添加幂等,默认10秒内不重复发送

2024/04/20

1.升级禁用直链规则为路由规则,内部判断变得十分复杂,有一些历史遗留和可能存在一些优先级问题,请自行测试

2.如果走了转码可以选择最高码率来切回直链,
也可反向操作,
~~被路由规则指定过的不可反向操作,部分播放器可能已经初始化为了HSL播放环境,~~
不要用自动码率,emby的判断有时不准确

3.客户端不支持的音频(Web端DTS)也会导致走转码,可以关闭用户的[播放-如有必要，在媒体播放期间允许视频转码]

4.不用转码功能的,保持[transcodeConfig.enable=false]

2024/04/19

1.升级分组禁用直链规则,添加NJS事件日志,重构拆分转码方法

2.经热心网友协助测试,发现拖动进度条黑屏属于emby的ios官方客户端和infuse播放115独有的问题,阿里云盘不存在此问题,
故升级了直链禁用规则,复现测试可以将115直链放到IDM等多线程下载器中,注意设置为获取直链时的UA,即使是115年费会员,
超过2个线程加载,就会导致整个下载进度卡住,时断时续,猜测这两个客户端默认使用了并发分块加载,
如果有相关设置的话(一般没有,是播放器内核自己的行为),自行设置为1,或者放开禁用直链规则

3.115还有一个问题,很久之前做文件迁移时发现的,一个账号不论多少客户端总共的下载进程最大10个,且有日传输量限制,
这个具体多少流量不清楚,复现测试为浏览器开10个任务,官方客户端下载文件夹只算一个任务,就会发现其余客户端无法播放和下载了

2024/04/16

1.!!!实验功能,转码负载均衡,默认false,将按之前逻辑禁止转码处理并移除转码选项参数,此条与emby配置无关,
主库和所有从库给用户开启[播放-如有必要，在媒体播放期间允许视频转码]+[倒数7行-允许媒体转换],
````
type: "nginx", // nginx负载均衡
````
好处是使用简单且内置均衡参数选择,且因为服务组只能局域网,
理论上对网盘挂载比较安全,因为都是同一出口IP,缺点是流量全部经过此服务器,
且使用条件很苛刻,转码服务组中的媒体id需要和主媒体库中id一致,自行寻找实现主从同步,完全同步后,ApiKey也是一致的

1.1 使用教程,遵循上述前置条件,
更改constant.js中transcodeConfig的enable属性为true,
type改为nginx,在nginx的server-group.conf负载均衡中配置转码服务组,
组内只支持局域网访问,且主从媒体服务类型需一致

2.分布式媒体服务负载均衡(暂未实现均衡)
````
type: "distributed-media-server",
````
优先利用302真正实现流量的LB,且灵活,不区分主从,当前访问服务即为主库,可emby/jellyfin混搭,
挂载路径可以不一致,但要求库中的标题和语种一致且原始文件名一致,
缺点是如果组内都是异网IP,对网盘安全性相对差一些,少量几个人用的无风险,
担心此问题的,把组内所有服务放在同一局域网保证出口IP一致,公网使用不同的端口映射,
虽然是302但是真正访问网盘是目标服务,因为要做转码,不是客户端访问

2.1 使用教程,前置条件只需要给访问的服务用户开启,目标服务可能不用开,使用的是ApiKey调用,没有用户,
[播放-如有必要，在媒体播放期间允许视频转码] + [倒数7行-允许媒体转换],
更改constant.js中transcodeConfig的enable属性为true,
type改为distributed-media-server,在下面的server中配置转码服务组,
组内需要所有服务公网畅通,因为需要客户端302到对应服务

3.实现原理

3.1 查询当前库媒体的名称标题和挂载路径并截取出带后缀的文件名

3.2 调用相同和兼容的查询接口从目标转码服务获取信息并过滤匹配,
首选使用名称标题查询,标题是库中显示的刮削匹配后的名称,
一般和tmdb或imdb对应语言标题相同,此过程和搜索框接口一致,
匹配不上的可以自己用搜索框尝试一下,获取到多个结果后再使用带后缀的文件名进行最终精确匹配

3.3 构造异构系统的转码访问地址并做一些少量的参数兼容转换,LB后302到目标转码服务

4.目前需要关注的注意点

4.1 控制台和播放详情页中的信息不准确,只显示原始码率信息和错误显示的直接播放,无法兼容,跨服务且转码参数过多

4.2 不兼容第三方播放器,因为没有码率选项和转码参数

4.3 虽然支持 emby <=> jellyfin 双向访问,
但简单测试发现部分情况下 jellyfin => emby 会有无法终止转码的风险,
例如在jellyfin没有发出停止播放和终止转码的情况下,emby无从得知是否该结束转码,
复现步骤为播放后直接刷新浏览器,一定要注意避免此种情况,耐心等待正常的播放和停止,
包括提示没有兼容的流也会同步终止转码,如发现emby没有终止转码,硬件占用高,
需要手动清空转码临时目录才能强制终止转码过程,控制台没有相关提示和操作,
emby => jellyfin 不存在此问题,jellyfin会在终止连接后自动且迅速的终止转码并清理临时目录

4.4 emby/jellyfin 混合配置的,注意目标emby必须用此项目反代才能同步播放开始结束和终止转码,
以免出现4.3的情况,当然测试情况下可以随意,直接配置原始emby服务也行,
同样的,目标服务为jellyfin的情况,可以直接配置原始服务

4.5 暂时没有对转码参数进行更改,全靠emby/jellyfin同源,不过可能还是存在部分参数不兼容的情况

4.6 如果当前访问服务为https,则目标服务必须全部使用https,不然302会被浏览器默认拦截,但在一些客户端不存在此问题

4.7 测试条件有限,只测试了 emby <=> jellyfin 的web端双向访问,且由于个人NAS核显等同于没有,
只能选择144P和240P码率互相切换做测试,其他码率的响应时间过长,
媒体服务客户端基本都是10秒内一个切片超时就会主动断开连接重新请求,导致几分钟内还没开始播放,
且只能依靠CPU 100% 和转码临时目录文件判断成功情况

4.8 只测试了CD2挂载alist阿里云盘的文件,理论上NAS本地文件转码响应会更快,因为没有缓存下载这个过程,
~~暂未测试STRM的支持情况~~
STRM会导致jellfin的PlaybackInfo从几秒增长到30秒,效率极差,不建议对STRM使用转码功能

4.9 可以和直链302共存,因为走的是不同的接口,但要保证选择最大码率或用修改版客户端开直连模式,此点基本和以前注意如何避免触发转码类似

4.10 只实现了流量的分离,暂未实现真正的负载均衡,目前极有可能会全部分配到第一台服务器,
因为没找到真正查询服务组转码数量的接口,后期可能参考nginx实现方式,NJS添加计数器解决,
但会出现新问题,分布式需要各节点信息共享,且转码数量不精确,没经过nginx处理的转码会统计不到

2024/04/13

1.注意需要保证njs >= 0.8.0,直接nginx:latest即可,
加入防抖措施,添加内外部重定向地址缓存,以兼容部分客户端不遵循30X重定向规范导致的短时过多重复请求穿透到alist,
例如emby安卓客户端,infuse,表现形式为每次拖动进度条都会重复请求emby原始串流地址,忽略上次拿到的重定向后地址,
Web端没有此问题,调用外部第三方播放器也无此问题,但是播放nas本地视频也会多次请求,故强制兼容解决,
可以打开embyRedirectSendMessage.enable = true,在官方客户端内进行提示查看,
默认按阿里云盘的直链最大有效时间15分钟,请勿随意更改此时间

2024/04/12

1.添加定时任务默认7天自动清空nginx日志,请结合日志重要程度和硬盘占用情况自行调整为合适间隔,建议不要改为小于1天以免影响性能,
使用条件为没有更改过默认日志的路径和名称,且需要更新最新版本njs
````
/var/log/nginx/error.log
/var/log/nginx/access.log
````

2.添加限流配置示例,只对302之前的请求生效,302后是直连第三方服务器,无法进行控制

2024/04/11

1.当媒体服务中存在过多的媒体时访问首页很慢,优化方式为,设置=>服务器=>数据库=>数据库缓存大小（MB）=>进行适当调大,
个人目前为460MB,请根据物理机内存情况合理设置,其他数据库设置请勿更改

2.添加nginx对接日志中心示例配置,可以和原xxx_log共存,如有需要,打开注释并修改为自己的ip和端口即可
发送日志到syslog,默认使用UDP的514端口,群晖=>日志中心=>接收日志=>新增=>名称随意,保持默认的BSD格式,UDP,514

2024/04/10

客户端绕过nginx-emby的参考,新代码已经对系统信息接口反代修改了端口号,不确定是否还有问题

1.之前遇到过类似的,但不完全相同,媒体服务都是有UDP广播局域网自动发现的,
但是广域网走的是服务开放的接口获取服务器的ip以及端口信息,例如emby默认的8096端口,
我路由器映射出去也是8096,这就导致接入了网络发现接口的客户端会擅自连接8096端口,
而忽略连接nginx的反代端口8098之类的,尽管我是手动使用8098端口登录的也没用,我是用的官方emby安卓客户端

我这边的解决方案是将路由器端口映射为8096=>7096,路由器这边的设置,emby是不可能知道的,
表现形式为控制台端口还是8096,所以客户端在擅自连接8096不通时就会老实连接反代的8091了

猜测fileball接口接入比较完善了,使用了媒体服务提供的网络发现接口,而infuse并没有调用此接口,以用户手动填写的为准

2024/04/08

1.增强禁用直链的规则配置,docker环境需要注意此参数客户端地址($remote_addr),
nginx容器网络必须为host模式,不然此变量全部为内网ip,判断无效

2024/04/05

如何避免媒体服务器频繁进行整库扫描刮削,导致内存占用飙升且影响性能,并范围太大会触发网盘的熔断机制

1.部分网盘熔断恢复周期,期间会全部拒绝服务,阿里云盘大概为30分钟到1小时,115网盘则为1-2小时

2.简单方案,发现整库扫描情况立即重启媒体服务,会消停几小时,但治标不治本,之后还会继续重试,控制台和计划任务里不会显示,无法强制停止

3.alist一定要开启115网盘驱动的限制速率,它的限制比较严格,一扫库必定熔断,默认为2【限制所有 api 请求速率(1r/[limit_rate]s)】,
意味着2秒内只处理1个请求,个人设置为1也是没问题的,此为最小值,为0是没限制,缺点是整库扫持续时间会达到一周,但是不会触发熔断,比较安全,
阿里网盘的熔断机制相对宽松很多,除非一次性文件太多,但是截止当前,alist并没有当前网盘的限流参数

4.最简单,全部使用strm文件解决,没有播放的情况下默认不会进行刮削,emby对此兼容性较好,只在第一次播放后会将媒体流参数信息存入数据库,
下次播放将跳过分析过程,表现形式为秒播,pelx对此无法跳过分析过程,播放开始会长达6-8秒等待,以上为官方客户端,第三方客户端不会有分析过程

以下为猜想未经测试,仅供记录

1.限流可以放在nginx反代配置,流量路径为 nginx-媒体服务 => 媒体服务 => cd2/rclone => nginx-alist => alist

2.媒体库触发强制自动扫库的根本原因为媒体文件的最后修改时间 > 入库时的最后修改时间,有条件可以在nginx-alist这层做反代返回假的最后修改时间,
或者等待alist添加自定义配置最后修改时间功能,截止目前,并没有相关issue

3.从挂载层解决,查询rclone官方文档得知有缓存参数,此测试结果为会在自定义配置的缓存目录同目录结构下,优先生成一个缓存文件,
缓存文件属性显示大小等于原始文件,但是实际占用大小只为读取的文件大小,例如1G文件,被刮削视频头后,大概实际只占用40%,完全没读取过,只占用0KB,
但是此缓存文件最后修改时间为最后一次读写时间,文档中没有自定义配置的参数,如果想要Web控制台,可以再套一层cd2

2024/03/31

1.对接emby设备控制推送通知消息,目前只发送是否直链成功,范围为所有的客户端,通知目标只为当前播放的设备,如果客户端短时间内走了缓存,则不会触发通知

2024/03/30

1.增强路径映射功能,区分来源类型

2.strm文件内部目前建议为远程链接,这样emby会在第一次播放后进行补充媒体数据,例如媒体格式和字幕流信息,这样官方客户端兼容性更好

3.strm文件内部为/开头的相对路径的虽然强行兼容支持播放,但是好像官方客户端的播放记录和回传存在bug,备选使用

4.添加默认可选参数以支持issue建议的指定strm规则获取重定向后的地址进行转发,兼容局域网的strm链接

5.感谢 @sgpublic 提供的 alist sign 计算方案
~~5.alist >= 3.20的默认对直链开启了sign全部参数,属于额外验证,不接受token验证,~~
~~如果要兼容,性能会很差,需要多用token请求一次alist获取到直链和sign参数,解决方案两种~~

~~5.1.用/开头的路径,这样会用alistToken走fsGet接口一次获取最终直链返回,缺点是官方客户端字幕流不正常且播放记录不准确或者没有~~

~~5.2.nginx请求的alist建议关闭设置-全局-签名所有,将此alist部署为和nginx同一局域网,接口响应也会快很多,通常在200ms-2000ms之间,跨网络会更慢~~
~~如果对直链安全有介意,去掉此alist的公网端口映射,只在局域网使用,公网使用另行部署一个开启sign全部的alist~~

2024/03/28

1.添加基本的配置示例文件,若符合需求,更改内容并删除文件名后缀,复制文件到上一级目录覆盖原始文件即可,\emby2Alist\nginx\conf.d\constant.js

2024/03/18

1.优化请求alist的115直链获取逻辑,透传UA,减少一次302过程,以兼容媒体服务器https而alist为http默认被浏览器客户端强制改写导致的错误

2.~~115直链不在需要cilentSelfAlistRule参数,~~
但保留处理逻辑,有特殊需要可自定义配置

3.将items隐藏升级为按路径匹配规则并新增,更多类似,接口隐藏

2024/03/14

1.按规则隐藏搜索接口返回的items

2024/03/13

1.媒体服务器https后,如果alist没有https,且相同域名情况,http链接会被浏览器默认强制改写为https,导致115的处理的第一次302会失败

2.地址栏左边-此网站权限-不安全的内容-允许,或者浏览器设置-Cookie 和网站权限-不安全的内容-允许

3.非浏览器不存在此问题,例如第三方播放器,默认不会阻止,也可将alist套上证书解决此问题

2024/03/10

1.测试并修复本地视频兼容问题,意外发现http2对本地原始链接的视频在部分跨宽带网络阻断有帮助(电信->联通),如有相同情况请开启http2或者http3

2024/03/04

1.优化播放时减少一次媒体查询接口

2024/03/01

1.串流地址加入媒体文件名方便标识和字幕匹配

2.添加图片缓存策略可选配置项

2024/01/20

1.添加实验功能,转码分流

2023/12/31

1.115的302需要alist最新版v3.30.0,由于115直链并没有响应允许跨域标识,所以只能用客户端播放,测试emby所有官方客户端和第三方客户端支持跨域,~~不支持跨域的播放为Web浏览器...~~

~~2.115播放形式为响应302到原始alist链接,由alist再302一次到直链~~

3.Web浏览器被跨域拦截请使用拓展解决,该拓展有时不稳定,表现形式为开启状态,但是并没有添加跨域响应头,可以寻找类似拓展测试,或者多开关几次并增大urlRegex匹配范围确保成功添加自定义响应头后再使用

3.1
https://microsoftedge.microsoft.com/addons/detail/header-editor/afopnekiinpekooejpchnkgfffaeceko

````
CORS Support
匹配类型: 域名
匹配规则: cdnfhnfile.115.com
执行类型: 常规
头名称: access-control-allow-origin
头内容: *
````
![image](https://github.com/chen3861229/embyExternalUrl/assets/42368856/5a06f4c5-3a39-4117-87db-899e7a0b22a6)

3.2
https://microsoftedge.microsoft.com/addons/detail/modheader-modify-http-h/opgbiafapkbbnbnjcdomjaghbckfkglc
````
[
    {
        "respHeaders": [
            {
                "enabled": true,
                "name": "Access-Control-Allow-Origin",
                "value": "*"
            }
        ],
        "shortTitle": "1",
        "title": "CORS Support",
        "urlFilters": [
            {
                "enabled": true,
                "urlRegex": "*.115.com"
            }
        ],
        "version": 2
    }
]
````
![image](https://github.com/bpking1/embyExternalUrl/assets/42368856/3ea94076-829f-4542-88e2-3221b9a8c8f4)

4.添加部分可选配置项,对接emby/jellyfin通知管理员设置,方便排查直链情况

2023/10/02

1.支持strm文件的直链,下边第一种情况已做处理默认支持

有多种填写方式,一个strm文件内部只能有一行路径或者链接,具体可以参考emby官方文档,我这里只测试了两种情况,例如:

1-1:
从alist的根路径开始填写,注意不要包含embyMountPath这个参数的路径,特殊字符不用转义,代码内部已做处理
![image](https://github.com/bpking1/embyExternalUrl/assets/42368856/240f5aa5-a603-40b6-ab6f-ed7c1b13c806)

1-2:
直接填写直链,~~这种稍微有风险,不建议使用,而且携带密码会被浏览器拦截,~~
且重定向后的远程链接将被部分浏览器跨域限制,无法修复,emby的客户端直接获取到并请求地址
![image](https://github.com/bpking1/embyExternalUrl/assets/42368856/5e904160-717b-4b8c-abf6-e08a8756de35)

2.emby在扫库的时候不会刮削strm文件的元数据,只有在视频第一次播放的时候才会获取,这需要几秒钟的时间,且是根据这个文件本身的路径以及文件名来识别的,和strm内容无关,空文件也能刮削出来,在播放时将内部链接传给客户端自己请求

3.根据部分反馈看,1-1的相对路径方式可能存在进度跟踪不准确,且没有在播放完毕后自动标记完成,建议使用标准的第三方工具生成的样式1-2

2023/09/28

1.实现客户端直链下载

2023/2/2

升级到alist v3了,脚本github地址 [bpking1/embyExternalUrl (github.com)](https://github.com/bpking1/embyExternalUrl)

调用外部播放器的油猴脚本账号无法登陆了,换了个新地址:[embyLaunchPotplayer (greasyfork.org)](https://greasyfork.org/en/scripts/459297-embylaunchpotplayer)

2022/5/13

1.兼容jellyfin

2.解决infuse无法播放的问题,感谢@amwamw968

3.用nignx添加了字幕文件的缓存支持

2022/1/12

1.重写了js脚本,支持rclone union,不再需要挂载名和alist盘名一致,不再需要设置emby api_key

2.修复了js脚本不能正确获取同一个视频不同清晰度版本的问题

2021/12/06

1.alist项目作者最近的更新中加入了阿里云盘之外的网盘的支持,且不在需要刷新目录

2.换了另外一个用rust语言写的阿里盘webdav项目,内存占用很小

3.修改了njs脚本中的正则,来修复emby魔改客户端terminusPlayer没有走直链

4.修改nginx配置修复了阿里云盘直链无法在emby web中播放

5.修复了由于反代无法使用jellyfin-mpv-shim直链播放

6.用nignx添加了emby图片的缓存支持

## 这篇文章的受众:
写这篇文章默认读者是emby用户,使用rclone挂载网盘,会使用docker,因篇幅问题以上软件的使用方法不在文章范围之中,此项目不会对原有的emby和rclone配置造成影响或修改

## 原理:
~~使用[aliyundrive-webdav](https://github.com/messense/aliyundrive-webdav) 项目将阿里盘转为webdav, 再~~
使用rclone挂载以供emby读取
使用[alist项目](https://github.com/Xhofe/alist) 将阿里盘及别的网盘的文件转为直链,使用nginx及其njs模块将emby视频播放地址劫持到 alist直链 
(~~暂时只测试了od,gd和阿里云盘可用,~~
alist目前支持好几种网盘,感兴趣的可以测试一下)

## 步骤:

### 1.先将配置文件下载到本地

```bash
wget https://github.com/bpking1/embyExternalUrl/releases/download/v0.0.1/emby2Alist.tar.gz && mkdir -p ~/emby2Alist && tar -xzvf ./emby2Alist.tar.gz -C ~/emby2Alist && cd ~/emby2Alist
```

此时文件结构如下:
~/emby2Alist
├── docker
|   ├── docker-compose.yml
└── nginx
    ├── conf.d
    │   ├── emby.conf
    │   └── emby.js
    └── nginx.conf

### 2. 
看情况修改constant.js 中的设置项目,通常来说只需要改alist密码
这里默认emby在同一台机器并且使用8096端口,~~否则要修改 emby.js和emby.conf中emby的地址~~
### 3 . 如果不挂载阿里云盘 可以跳过这一步
修改docker-compose.yml 中 service.ali-webdav 的 REFRESH_TOKEN
获取方法参考原项目地址: https://github.com/messense/aliyundrive-webdav

### docker部署的任选以下一种
xxx为示例目录名,请根据自身情况修改

前置条件1: 需要手动创建目录
````
/xxx/nginx-emby/log
/xxx/nginx-emby/embyCache
````
前置条件2: 需要手动移动项目配置文件
````
将本项目xxx2Alist/nginx/下所有文件移动到/xxx/nginx-emby/config/下面
````

### 4.1 - docker-compose
启动服务: 在 ~/emby2Alist/docker 目录下执行
```bash
docker-compose up -d
```
查看启动log:
```bash
docker-compose logs -f
```
如果log有报错,请按照提示信息修改 ,常见错误可能为
1. docker端口占用冲突:  修改docker-comopse映射端口
2. webdav 的refresh token 填写错误 (**如果不挂载阿里云盘则忽略**)

### 4.2 - 群晖docker
容器=>设置=>导入=>选择json配置文件=>确认

### 5. 
防火墙放行 5244, 8091 ~~和 8080端口~~
8080端口为阿里盘 webdav地址 , 8091 端口为emby转直链端口与默认的8096互不影响
访问5244端口,初始密码查看docker log能看到 ,根据项目文档 https://github.com/Xhofe/alist 在Alist项目后台添加网盘 
注意: 

1. 添加od,gd盘可以直接复制rclone配置里面的 clientid , secret , refreshToken,不用再麻烦去重新搞一次了
2. **不使用阿里云盘可以跳过这步**
   alist阿里盘的refreshToken与webdav那个token是不一样的,这里需要的是要不需要referrer请求头的token,详情请参考这个[issue](https://github.com/Xhofe/alist/issues/88) , 可以用这个网页来获取 [阿里云盘 - RefreshToken (cooluc.com)](https://media.cooluc.com/decode_token/) 
3. 盘名建议一致,这样获取直链更快,不一致也可以

~~添加的网盘在alist里面的名称需要与 rclone挂载的文件夹名称一样  比如挂载路径为 /mnt/ali 那么盘的名称也要叫 ali~~

### 6. 如果不挂载阿里云盘 可以跳过这一步
配置rclone,挂载网盘,这里以阿里盘webdav为例

使用rclone 挂载 阿里盘webdav 
第一步name  我这里为 ali
rclone config  选 webdav , 地址为http://localhost:8080 默认用户和密码都为admin
rclone lsf ali:  看一下能否获取到列表
创建文件夹:
mkdir -p /mnt/ali     注:此挂载文件夹的名字需要与 Alist 中的盘名相同
挂载:

```bash
nohup rclone mount ali: /mnt/ali --umask 0000 --default-permissions --allow-non-empty --allow-other --buffer-size 32M --vfs-read-chunk-size 64M --vfs-read-chunk-size-limit 1G &
```
也可以写成service


### 7.

访问 8091 端口打开emby 测试直链是否生效,查看执行log
```bash
docker logs -f -n 10 emby-nginx 2>&1  | grep js:
```
8091端口为走直链端口  , 原本的 8096端口 走 emby server 不变
直链播放不支持转码,转码的话只能走emby server
所以最好 在emby设置中将 播放 --> 视频 --> 互联网质量 设置为最高 ,并且将用户的转码权限关掉,确保走直链
web端各大浏览器对音频和视频编码支持情况不一,碰到不支持的情况emby会强制走转码而不会走直链

## 已知问题:
1. emby web播放时如果需要使用内封的字幕,实际上是需要embyServer在后台用ffmpeg去提取的,~~ffmpeg要读取整个视频文件才能获取所有的字幕流,相当于几乎整个视频文件都要通过rclone下载,并且消耗cpu资源,对于比较大的视频文件是不现实的,所以web端建议使用外挂字幕~~,从头读取到字幕流位置截止,大概占文件大小的40%. 只有修改版emby客户端调用MX Player会同时传递所有外挂字幕,其余方式包括串流地址不支持外挂字幕加载,需要手动下载字幕文件并选择装载
2. ~~google Drive由于api的限制直链只能通过server中转,所以还是建议在cf上搭建goindex来获取直链 ,如何给到emby请参考 这篇[文章](https://blog.738888.xyz/2021/09/09/emby%E6%8C%82%E8%BD%BD%E7%BD%91%E7%9B%98%E8%BD%AC%E7%9B%B4%E9%93%BE%E6%92%AD%E6%94%BE/)结尾,另外一种方法是给alist添加cf worker中转gd的支持,有待研究~~
alist新版已经支持cf worker代理gd下载了,详情参考alist文档
3. 可能会有其他问题,请留言
