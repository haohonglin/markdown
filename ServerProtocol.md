简介
====

* App 里有三部分会和服务端通信：账号系统，论坛，静态内容
    账号系统：
        首次打开 App 会向服务器请求创建一个账号，随机用户名及密码。
        以后用户可以选择绑定邮箱并修改密码。
    论坛：
        就是个简单的论坛。按照节目划分板块。
    静态内容：
        每个节目会有若干条静态内容。对每条静态内容，用户可以选择收藏或评论。
* 我们希望静态内容及评论在节目播出后的特定时刻移动至论坛。每条静态内容及其评论
    构成一个主题。所以把静态内容的评论做成论坛中特殊的主题帖可能比较好。
* 主题分两种，一种叫“静态内容主题”（info thread），一种叫“论坛主题”（forum
    thread）。前者对应静态内容（新闻、投票等）的评论，楼主是管理员，标题和
    主贴内容为空。后者为论坛中用户发表的帖子。

* URL 不对返回 404
* METHOD 不对返回 405
* 所有 POST 方法都需要验证 session（/register 除外），验证失败返回 403，下边不再重复
* query 的必需参数缺少时返回 404，参数格式/值非法时返回 400

* 一些常数，可调整
    PASSWORD_LENGTH_MIN = 6
    THREAD_PAGE_SIZE = 20
    REPLY_PAGE_SIZE = 30
    CONTENT_LENGTH_MAX = 10000
    REPLY_LENGTH_MAX = 100 


* 返回值时 json 时 Content-Type 设为 application/json; charset=UTF-8
* 返回值里的时间戳都是自 Unix epoch 的毫秒数，所有时间都是 UTC

* 下列API分成三个部分, 每部分对应一个Domain, API和Domain的对应关系请看文档api_domain_map.txt
* 我们在下列API上加上前缀/v1/作为 version 1.0 的API

* server 主要有以下几个模块：

1. [帐号](#帐号)
2. [静态内容](#静态内容)
3. [用户反馈](#用户反馈)
4. [论坛](#论坛)
5. [投票](#投票)

---


账户
====
邮箱注册
---

```
URL: /register_email
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    email
    nick
    gender          Int     1 或 2
    newpassword
    device          String  可选，设备 ID
    device_model    String  可选，设备型号
Action: 注册邮箱，生成新的 session
    (1) 邮箱已被使用
        email 格式非法
        nick 太短，或格式不合法（格式要求待定，如不能以 "(微博用户)" 或 "(微信用户)" 结尾）
        密码少于 PASSWORD_LENGTH_MIN 个字符
        response: 200
        data example:
            {'error':'xxxx', 'error_msg':'xxxxx'}
    (2) nick 与其他用户重复
        email 已被其他用户绑定
        response: 200
        data example:
            { 'error': '20012', 'error_msg': '具体错误信息'  }
    (3) 成功
        response: 200
        data example:
        data 同 /v1/login_email 返回值
```
---
邮箱登录
---
```
URL: /login_email
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    email
    password
    device          String  可选，设备 ID
    device_model    String  可选，设备型号
Action: 使用邮箱及密码登陆
(1) 成功
    response: 200
    data example:
    {
        'username':'0123456789',
        'session':'xxxxxx',
        'nick':'张三',
        'profile_image':'http://xx.xx/xx.png',
        'profile_banner':'http://xx.xx/yy.png',
        'email':'xx@xx.xx',
        'weibo_uid':'2000123123',
        'weibo_name':'微博名',
        'weixin_uid':'1300000000',
        'weixin_name':'微信名'，
        'birthday':'yy-mm-dd',
        'gender': 0
        'location':'广东 珠海'
        'signature':'啦啦啦'
    }
    properties:
        username    String(10)      10 位数字用户名
        session     String          若干位 session ID
        nick        String          海米号，显示出来的用户昵称
        birthday    String          用户生日
        gender      int             用户的性别 (1: 男, 2: 女， 非1和2：未知)
        location    String          用户所在地
        sigture     String          用户签名
        profile_image   String      头像 URL，可选
        profile_banner  String      头像背景图片 URL，可选
        email       String          绑定的邮箱地址，可选
        weibo_uid   String          绑定的微博用户 id，可选
        weibo_name  String          绑定的微博用户名，可选
        weixin_uid  String          绑定的微信用户 id，可选
        weixin_name String          绑定的微信用户名，可选
```
---
微博登录
---
```
URL: /login_weibo
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    uid                     微博用户 id
    token
    token_expire    Long    token 过期时间
    screen_name             微博用户名
    profile_image           微博头像地址
    device          String  可选，设备 ID
    device_model    String  可选，设备型号
Action: 使用微博帐号登录
(1) 成功
    response: 200
    data 同 /v1/login_email 返回值
```
---
微信登录
---
```
URL: /login_weixin
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    code            String
    device          String  可选，设备 ID
    device_model    String  可选，设备型号
Action: 使用微信帐号登录，返回值同/v1/login_weibo
```
---
邮箱绑定
---
```
URL: /bind_email
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    username
    session
    email
    nick
    newpassword
Action: 为用户绑定邮箱，生成新的 session
    (1) 指定用户已绑定邮箱
        email 格式非法
        *nick长短限制有必要吗*
        nick 太短，或格式不合法（格式要求待定，如不能以 "(微博用户)" 或 "(微信用户)" 结尾）
        密码少于 PASSWORD_LENGTH_MIN 个字符
        response: 200
    (2) nick 与其他用户重复
        email 已被其他用户绑定
        response: 200
        data example:
            { 'error': '20012', 'error_msg': '具体错误信息'  }
    (3) 成功
        response: 200
        data 同 /v1/login_email 返回值
```
---
微博绑定
---
```
URL: /bind_weibo
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    username
    session
    uid                     微博用户 id
    token
    token_expire    Long    token 过期时间
    screen_name             微博用户名
    profile_image           微博头像地址
    ...                     其他微博用户信息，待添加
Action: 绑定微博账号
    若用户尚未绑定其他账号，则将 nick 设置为 screen-name，并设置 profile_image
    若 screen_name 和已有用户 nick 重复，则将 nick 设置为 "<screen_name>(微博用户)"
    (1) 指定用户已绑定微博
        uid, token 格式非法(目前只判断是否为空)
        response: 200
        data example:
            { 'error': '20012', 'error_msg': '具体错误信息'  }

    (2) 成功
        response: 200
        data example:
        返回值和/v1/login_weibo 相同
```
---
微信绑定
---
```
URL: /bind_weixin
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    username
    session
    code
Action: 绑定微信
    返回值跟/v1/bind_weibo 一样
```
---
微博解绑
---
```
URL: /unbind_weibo
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    username
    session
Action: 解绑微博账号
    (1) 成功
        response: 200
```
---
微信解绑
---
```
URL: /unbind_weixin
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    username
    session
Action: 解绑微信账号
    (2) 成功
        response: 200
```
---
用户信息更新
---
```
URL: /update_userinfo
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    username
    session
    gender              可选
    birthday            可选
    location            可选
    signature           可选
Action: 更新用户信息，其中包括 gender, birthday, location, signature
(1) 成功
    response: 200
```
---
用户信息查询
---
```
URL: /userinfo
Domain: api.dev.hrmes.tv/v1/users
Method: GET
Parameters:
    username
    me      请求发起者的 username (可选)
Action: 获取任一用户信息
(1) 成功
    response: 200
    data example:
    {
        'birthday': 'yyyy-mm-dd',
        'gender': 0,
        'location': '广东 珠海',
        'signature': '啦啦啦',
        'profile_banner': 'http://xx.xx/yy.png',
        'post_count': 123,
        'following_count' : 10       
        'followed_count':  10       
        'follow_me' : True         (可选)  
    }
    properties:
        birthday    String          用户生日
        gender      Int             用户的性别 (1: 男, 2: 女， 非1和2：未知)
        location    String          用户所在地
        sigture     String          用户签名
        profile_banner  String      头像背景图片 URL，可选
        post_count  Int             用户发帖数
        following_count         Int     关注用户数
        followed_count          Int     被关注用户数
        follow_me (可选)        Bool    该用户是否关注请求发起者（请求参数 me），未给定参数 me 时返回值无此字段
```
---
好友关系检查
---
```
URL: /check_friend
Domain: api.dev.hrmes.tv/v1/users
Method: GET
Parameters:
    username
    me      请求发起者的 username
Action: 判断两个人是否为好友
(1) 成功
    response: 200
    data example:
    {
        'is_friend' : True       
    }
    properties:
        is_friend (可选)        Bool      表示两个用户是否为好友
```
---
找回密码
---
```
URL: /getback_password
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    email
Action: 找回密码， server 发送新密码到用户email
(1) email无效
    找不到指定email，（该email尚未绑定或注册）
    data example:
        {'error':'xxxx', 'error_msg': 'xxxxx'}
(2) 成功
    response: 200
    data example:
        {}
```
---
重置密码
---
```
URL: /reset_password
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    username
    session
    email
    oldpassword
    newpassword
Action: 更改用户的密码，并更新和返回user的session
(1) email无效
    oldpassword 与原来的密码不同
    newpassword 格式非法
    response: 200
    data example:
        {'error':'xxxx', 'error_msg': 'xxxxx'}
(2) 成功
    response: 200
    data example:
        {'session':new session}
```
---
个推绑定
---
```
URL: /bind_push
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameters:
    username
    session
    push_user
    push_channel
Action: 绑定推送信息，以接收单点推送
(1) 成功
    response: 200
```
---
关注
---
```
URL: /follow
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameter:
        username
        session
        target              String          被关注用户的 username
Action: 关注 username 为 target 的用户
Response:
    200，即使 target 用户已经被关注了
```
---
取消关注
---
```
URL: /unfollow
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameter:
        username
        session
        target              String          被取消关注用户的 username
Action: 取消关注 username 为 target 的用户
Response:
    200，即使 target 用户并没有被关注
```
---
关注列表
---
```
URL: /following_list
Domain: api.dev.hrmes.tv/v1/users
Method: GET
Parameter:
        username            String      目标用户
        me (可选)           String      发起请求的用户 username
        start (可选)        String      起始位置，上一页最后一条关注关系的 ID
        get_all (可选)      Int         为 1 时返回整个列表，否则根据 start 参数分页。默认分页。
Action: 获得指定用户的关注列表，按创建时间从新到旧排序
Response:
    Example:
        {
            "users": [{
                "id": "ABCDEF12345678",
                "username": "12345678",
                "nick": "yyy",
                "profile_image": "http://...",
                "follow_me": true        
                }, ...],
            "more": true                      
        }
        Properties:
            users                       用户列表
            id                      关注关系 ID
            username                用户名 
            nick                    昵称
            profile_image (可选)   mZ用户头像            
            follow_me (可选)        该用户是否关注请求发起者（请求参数 me），未给定参数 me 时返回值无此字段
            more                        之后是否有更多
```
---
被关注列表
---
```
URL: /followed_list
Domain: api.dev.hrmes.tv/v1/users
Method: GET
Parameter:
        username            String      目标用户
        me (可选)           String      发起请求的用户 username
        start (可选)        String      起始位置，上一页最后一条关注关系的 ID
Action: 获得指定用户的被关注列表，按创建时间从新到旧排序
Response:
        格式同 /following_list
```
---
关系查询
---
```
URL: /relation
Domain: api.dev.hrmes.tv/v1/users
Method: GET
Parameter:
        username            String      目标用户
        me           String      发起请求的用户 username
Action: 获得username 和 me 的关系
Response:
        json 字符串
        
        {
            'follow' : Bool,   #True 表示me关注了username
            'followed': Bool,  # True 表示username 关注了 me
        }
```
---
好友列表
---
```
URL: /friends
Domain: api.dev.hrmes.tv/v1/users
Method: GET
Parameter:
        username
Action: 获得指定用户的好友列表（同时关注和被关注）
Response:
Example:
        {
        "users": [{
            "username": "12345678",
            "nick": "yyy",
            "profile_image": "http://...",    
            }, ...]    
        }
        Properties:
            users                       用户列表
            username                用户名 
            nick                    昵称
            profile_image (可选)    用户头像
```
---
个推id绑定
---
```
URL: /upload_pushclientid
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameter:
    username
    session
    clientId    个推给用户分配的id

Action: 上传用户对应的个推client id
    (1) 成功
        response: 200
```
---
```
URL: /query_unread_count
Domain: api.dev.hrmes.tv/v1/users
Method: GET
Parameter:
        username
        session

Action: 上传用户对应的个推client id
    (1) 成功
        response: 200
        content_type: application/json
        data:
            {
                'reply':1,
                'follow':1,
                'like':10,
            }
        properties:
            'reply'     回复通知未读消息数
            ‘follow’    粉丝通知唯独消息数
            ‘like’        点赞通知唯独消息数
```
---
```
URL: /delete_notification
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameter:
        username
        session
        id             消息的id

Action: 上传用户对应的个推client id
    (1) 成功
        {response:200}
```
---
```
URL: /messages/follow
Domain: api.dev.hrmes.tv/v1/users
Method: GET
Parameter:
    username
    session
    start       String          可选，当前最新消息的 ID，返回比start更新的消息
    end         String          可选，当前最旧消息的 ID，返回比end更旧的消息
                                start 与 end 不能同时出现

Action: 获取用户的消息，按时间倒序排列。目前返回全部消息，以后会改为仅返回最新的若干条。
(1) 成功
    p 不合法返回 403
    start 越界返回 []
    response: 200
    content_type: application/json
    data:
        {
            more: True,
            messages:[
                {
                    'username': 'xxx',
                    'nick':'xxx', 
                    'profile_image':'http://xxxx.jpg',
                    'id': 'deadbeef12345678',
                    'detail': '好顶赞',
                    'rstatus': 0,
                    'time': 12300000000,
                },
            ]
        }
    properties:
        username     回复用户的username
        profile_image     回复用户的头像
        nick     回复用户的昵称
        id              消息 ID
        detail          消息内容
        rstatus       消息是否已读 （0表示未读，1表示已读）
        time            该消息产生时间
```
---
```
URL: /messages/reply
Domain: api.dev.hrmes.tv/v1/users
Method: GET
Parameter:
    username
    session
    start       String          可选，当前最新消息的 ID，返回比start更新的消息
    end         String          可选，当前最旧消息的 ID，返回比end更旧的消息
                                start 与 end 不能同时出现

Action: 获取用户的消息，按时间倒序排列。目前返回全部消息，以后会改为仅返回最新的若干条。
(1) 成功
    p 不合法返回 403
    start 越界返回 []
    response: 200
    content_type: application/json
    data:
        {
            more: True,
            messages:[
                {
                    'username': 'xxx',
                    'nick':'xxx', 
                    'profile_image':'http://xxxx.jpg',
                    'id': 'deadbeef12345678',
                    'type': 'messageType',
                    'quote': '创新城邦',
                    'detail': '好顶赞',
                    'rstatus': 0,
                    'p': '87654321deadbeef',
                    't': '87548201afa77219',
                    'r': 'deadbeef1234cafe',
                    'time': 12300000000,
                },
            ]
        }
    properties:
        username     回复用户的username
        profile_image     回复用户的头像
        nick     回复用户的昵称
        id              消息 ID
        type            消息类型，目前只有三种：replyInfo， threadForum 和 replyForum，
                        分别表示在内容详情页被回复和在讨论区被回复
        quote           回复所在位置的标题，
                        对于 replyInfo 类型，为该同步内容标题，
                        对于 replyForum 类型，为该节目(program)名称
        detail          回复正文
        rstatus       消息是否已读 （0表示未读，1表示已读）
        p               论坛版面 ID
                        对于 replyInfo 类型，为所属节目当期 ID (episode ID)
                        对于 replyForum 类型，为节目总 ID (program ID)
        t               帖子 ID
                        对于 replyInfo 类型，为该节目内容 ID
                        对于 replyForum 类型或者threadForum，为所在主题帖 ID
        r               回复 ID
        time            该消息产生时间
```
---
```
URL: /messages/like
Domain: api.dev.hrmes.tv/v1/users
Method: GET
Parameter:
    username
    session
    start       String          可选，当前最新消息的 ID，返回比start更新的消息
    end         String          可选，当前最旧消息的 ID，返回比end更旧的消息
                                start 与 end 不能同时出现

Action: 获取用户的消息，按时间倒序排列。目前返回全部消息，以后会改为仅返回最新的若干条。
(1) 成功
    p 不合法返回 403
    start 越界返回 []
    response: 200
    content_type: application/json
    data:
        {
            more: True,
            messages:[
                {
                    'username': 'xxx',
                    'nick':'xxx', 
                    'profile_image':'http://xxxx.jpg',
                    'id': 'deadbeef12345678',
                    'type': 'messageType',
                    'quote': '创新城邦',
                    'detail': '好顶赞',
                    'rstatus': 0,
                    'p': '87654321deadbeef',
                    't': '87548201afa77219',
                    'r': 'deadbeef1234cafe',
                    'time': 12300000000,
                },
            ]
        }
    properties:
        username     回复用户的username
        profile_image     回复用户的头像
        nick     回复用户的昵称
        id              消息 ID
        type            消息类型，目前只有两种：replyInfo, threadForum 和 replyForum，
                        分别表示在内容详情页被回复和在讨论区被回复
        quote           回复所在位置的标题，
                        对于 replyInfo 类型，为该同步内容标题，
                        对于 replyForum 类型，为该节目(program)名称
        detail          回复正文
        rstatus       消息是否已读 （0表示未读，1表示已读）
        p               论坛版面 ID
                        对于 replyInfo 类型，为所属节目当期 ID (episode ID)
                        对于 replyForum 类型，为节目总 ID (program ID)
        t               帖子 ID
                        对于 replyInfo 类型，为该节目内容 ID
                        对于 replyForum 类型或者threadForum，为所在主题帖 ID
        r               回复 ID
        time            该消息产生时间
```

静态内容
========
```
URL: /favor
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameter:
    username
    session
    *change to list*
    items   json.dumps format of list     '[{"t": 主题ID, "time": 收藏时间戳}, {"t": xxx, "time": xxx}]'
Action: 收藏指定静态内容
(1) 成功
    response: 200
```
---
```
URL: /unfavor
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameter:
    username
    session
    t               主题 ID
Action: 取消收藏指定静态内容
(1) 成功
    response: 200
```
---
```
URL: /favorite
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameter:
    username
    session
Action: 返回指定节目的收藏列表
(1) 成功
    response: 200
    data example:
        [{ p: 5, t: 10, time: 1024000 }, ...]
    properties:
        p       节目 ID
        t       主题 ID
        time    收藏时间戳
```

用户反馈
=========
```
URL: /feedback
Domain: api.dev.hrmes.tv/v1/users
Method: POST
Parameter:
    username
    session
    feedback    String(length < 100)
Action: 用户反馈意见
(1) 成功
    response: 200
```
---
```
URL: /search_user
Domain: api.dev.hrmes.tv/v1/users
Method: GET  
Parameter:
    keyword                 String     搜索的关键词
    pagenum     (optional)  String  请求的第几页数据（默认值为0，0代表第一页）

Action: 根据关键字获取相应的用户信息，按精确匹配，首字母排序，我已关注的ID,关注我的ID
Response:
    Example:
    {
        "users": [{
            "username": "12345678",
            "nick": "yyy",
            "profile_image": "http://...",
            "relation": 1
        }, ...],
        "more": true
    }
Properties:
    users                       用户列表
         username                用户名
         nick                    昵称
         profile_image           用户头像
         relation                0 = 无关注关系 ， 1 = 我已经关注的ID , 2 = 关注我的ID , 3 = 双向关注关系
    more                        之后是否有更多
```

论坛
====
```
URL: /new_program
Domain: api.dev.hrmes.tv/v1/maintains
Method: POST
Parameters:
    username            管理员帐号用户名
    password
    name                节目名称
    parent              可选。为整个节目创建 ID 时为空，为某一期节目(episode) 创建 ID 时为整个节目的 ID
Action: 创建新的节目，需要管理员账户和密码
(1) 成功
    response: 200
    properties:
        p:              节目 ID
注：需要管理员权限。
```
---
```
*全局的行为，不一定要和用户绑定，故不需要用户名和密码
URL: /new_thread
Domain: api.dev.hrmes.tv/v1/forums
Method: POST
Parameters:
    username
    session
    p               节目 ID
    content         帖子内容
Action: 发表新论坛主题帖子
(1) 节目不存在
    content 长度超过 POST_LENGTH_MAX
    response: 403
(2) 成功
    response: 200
    properties:
        t:              帖子 ID
```
---
```
URL: /new_info_thread
Domain: api.dev.hrmes.tv/v1/maintains
Method: POST
Parameters:
    username            管理员用户名
    session
    p                   节目 ID
    name                新建同步内容名称，用于生成该内容下用户间回复产生的消息
Action: 创建新的同步内容主题。
(1) 节目不存在
    response: 403
(2) 成功，返回同步内容 id
    response: 200
    data example:
        {'t': "xxx"}
注：需要管理员权限。
```
---
```
URL: /threads
Domain: api.dev.hrmes.tv/v1/forums
Method: GET
Parameters:
    p           Int32           节目 ID（即版面 ID）
    start       Int32           需要返回的第一个主题的编号，可选，默认为 0
Action: 返回指定主题指定帖子开始的 THREAD_PAGE_SIZE 个论坛主题。
    按最后回复时间排序。
(1) 成功
    p 不合法返回 403
    start 越界返回 []
    response: 200
    data example:
        [{
            't': '123456',
            'username': 'xxx',
            'nick': 'yyy',
            'profile_image' : 'http://xxxx.jpg',
            'content': 'zzz',
            'publish_time': 1407900902000,
            'reply_time': 1407900902000,
            'reply_count': 10,
            'like': 100,
            'replies': []
        }, ...]
    properties:
        t                           帖子 ID
        username                    主贴用户名
        nick                        昵称
        profile_image (可选)        主贴用户的头像
        content                     主贴内容
        publish_time                发表时间戳
        reply_time                  最后回复时间戳
        reply_count                 回复数
        like                        点赞数
        replies                     列表， 格式和/replies22相同
```
---
```
URL: /new_reply2
Domain: api.dev.hrmes.tv/v1/forums
Method: POST
Parameters:
    username
    session
    t               主题 ID
    content         帖子内容
    reply_to        被回复用户的 ID，可选
Action: 发表回复，并返回该条回复及之前相邻的若干条
(1)
    content 长度超过 POST_LENGTH_MAX
    response: 403
(2) 成功
    response: 200
    data example:
        {
            'replies': [...],
            'more': true
        }
    格式与 /replies2 相同，其中 more 表示返回的列表之前是否还有更多回复
```
---
```
URL: /replies2
Domain: api.dev.hrmes.tv/v1/forums
Method: GET
Parameters:
    t           String          主题 ID
    start       String          可选，需要返回的第一条帖子的 ID
    end         String          可选，需要返回的最后一条帖子的 ID
                                start 与 end 不能同时出现
    exclusive   Int32           可选，返回的列表是否包含指定 ID 的帖子。
                                1 表示“是”，其他值或不指定表示“否”
Action: 返回指定主题指定帖子开始或结束的 REPLY_PAGE_SIZE 条回复。
        当 start 和 end 均未指定时，返回最开始的若干回复，并忽略 exclusive 参数。
(1) 成功
    p 不合法返回 403
    start 越界返回 []
    response: 200
    content_type: application/json
    data example:
        {
            'replies': [{
                'r': '123456'
                'username': 'xxx',
                'nick': 'yyy',
                'profile_image': 'http://xxx.xxx/dd.jpg'
                'content': 'zzz',
                'publish_time': 1407900902000,
                'reply_to': 'yyy',
                'reply_to_nick': 'ZhangSan',
                'like': 100,
            }, ...],
            'more': true
        }
    properties:
        replies                     回复列表
            r                       回复 ID
            username                用户名
            nick                    昵称
            profile_image (可选)    用户头像
            content                 回复内容
            publish_time            发表时间戳
            reply_to (可选)         被回复用户 ID
            reply_to_nick (可选)    被回复用户昵称
            like                    点赞数
        more                        布尔值。
                                    无 start 参数但有 end 参数时，表示返回列表之前是否有更多回复。
                                    其他情况表示返回列表之后是否有更多回复。
``` 
---
```
URL: /replies/hot_new
Domain: api.dev.hrmes.tv/v1/forums
Method: GET  
Parameters:
    t           String          主题 ID
    start       String          可选，最后一条帖子的 ID
    exclusive   Int32           可选，返回的列表是否包含指定 ID 的帖子。1 表示“是”，其他值或不指定表示“否”
    flag         int            可选 （1 =代表 按时间排序 [也是默认排序方式]，2=最热的排序方式 ，3=最热排序且每次返回最多三条数据）
    pagenum      int            可选  获取最热的数据必须传人, （第几页的参数，默认为0)，该请求不需要传入start参数
    
Action: 获取最新和热的评论
    example:
           {
               'replies': [{
                'r': '123456'
                'username': 'xxx',
                'nick': 'yyy',
                'profile_image': 'http://xxx.xxx/dd.jpg',
                'content': 'zzz',
                'publish_time': 1407900902000,
                'reply_to': 'yyy',
                'reply_to_nick': 'ZhangSan',
                'like': 100,
            }, ...],
            'more': true
        }
        
properties:

        replies                     回复列表
            r                       回复 ID
            username                用户名
            nick                    昵称
            profile_image (可选)    用户头像
            content                 回复内容
            publish_time            发表时间戳
            reply_to (可选)         被回复用户 ID
            reply_to_nick (可选)    被回复用户昵称
            like                    点赞数
        more                        布尔值。
```
---
```
URL: /replies_and_thread2
Domain: api.dev.hrmes.tv/v1/forums
Method: GET
Parameters:
    t           String           主题 ID
    end         String          可选，需要返回的最后一条帖子的 ID
Action: 返回指定主题指定帖子结束的 REPLY_PAGE_SIZE 条回复以及主帖内容
(1) 成功
    data example:
        {
            'thread': {...}
            'replies': [...]
            'more': true
        }
    properties:
        thread          主贴内容，格式同 /threads2 返回值中的一项，其中的 replies 域为 []
        replies         指定回复列表，格式同 /replies2 返回值中回复列表
        more            回复列表之前是否有更多回复
```
---
```
URL: /like
Domain: api.dev.hrmes.tv/v1/forums
Method: POST
Parameter:
        username
        session
        t                  可选，帖子或静态内容 ID
        r                  可选，评论 ID。
Action: 对指定内容点赞。给参数 t 时对帖子或静态内容点赞，给参数 r 时对回复点赞。这两个参数有且只能有一个参数。
Response:
        (1) 返回 200，忽略重复点赞之类的错误。App 会忽略这个响应。

```
---
```
URL: /unlike
Domain: api.dev.hrmes.tv/v1/forums
Method: POST
Parameter:
        username
        session
        t                  可选，帖子或静态内容 ID
        r                  可选，评论 ID。
Action: 对指定内容取消点赞。参数要求同/v1/like
Response:
        (1) 返回 200
```

投票
==========
```
URL: /guess
Domain: api.dev.hrmes.tv/v1/votes
Method: POST
Parameter:
    username
    session
    t                   投票对应同步内容 ID
    choice              投票选项列表json字符串, 如 '["1", "2"]'
Action: 进行竞猜
(1) 成功，返回一个字典，表示当前投票结果
    response 200
    data:
    {
        'choice': ['a'],
        'result':
        {
            'a': 23,
            'b': 912,
            'c': 871
        }
    }
```
---
```
URL: /guess
Domain: api.dev.hrmes.tv/v1/votes
Method: GET
Parameter:
    username
    session
    t                   投票对应同步内容 ID
Action: 查询竞猜结果
(1) 成功，返回一个字典，表示当前投票结果
    response 200
    data:
    {
        'choice': ['a'],
        'result':
        {
            'a': 23,
            'b': 912,
            'c': 871
        }
    }
```
---
```
URL: /guess/all
Domain: api.dev.hrmes.tv/v1/votes
Method: GET
Parameter:
    username
    session
    tids                   字符串数组， 数组内容包含每个竞猜 ID, 如 '["123", "124"]'
Action: 查询竞猜结果
(1) 成功，返回一个字典，表示当前投票结果
    response 200
    data:
        {
            '123':{
                    'choice': ['a'],
                    'result':
                    {
                        'a': 23,
                        'b': 912,
                        'c': 871
                    }
                },
            '124':{
                    'choice': ['a'],
                    'result':
                    {
                        'a': 23,
                        'b': 912,
                        'c': 871
                    }
                }
        }
```
---
```
URL: /new_vote
Domain: api.dev.hrmes.tv/v1/maintains
Method: POST
Parameter:
    username
    session
    t                   对应同步内容 ID
    choices             JSON 格式字符串数组，投票选项列表
Action: 为某个同步内容创建对应投票
(1) 成功
    response 200
注：需要管理员权限。
```
---
```
URL: /vote
Domain: api.dev.hrmes.tv/v1/votes
Method: POST
Parameter:
    username
    session
    t                   投票对应同步内容 ID
    choice              投票选项列表json字符串, 如 '["1", "2"]'
Action: 进行投票
(1) 成功，返回一个字典，表示当前投票结果
    response 200
    data:
    {
        'choice': ['a'],
        'result':
        {
            'a': 23,
            'b': 912,
            'c': 871
        }
    }
```
---
```
URL: /vote
Domain: api.dev.hrmes.tv/v1/votes
Method: GET
Parameter:
    username
    session
    t                   投票对应同步内容 ID
Action: 进行投票
(1) 成功，返回一个字典，表示当前投票结果
    response 200
    data:
    {
        'choice': ['a'],
        'result':
        {
            'a': 23,
            'b': 912,
            'c': 871
        }
    }
```
---
```
URL: /vote_result
Domain: api.dev.hrmes.tv/v1/votes
Method: GET
Parameter:
    t                   投票对应同步内容 ID
Action: 获取某个投票的结果
(1) 成功
    response 200
    返回结果与 /v1/vote 相同
```
---
```
URL: /new_vote_cheer
Domain: api.dev.hrmes.tv/v1/maintains
Method: POST
Parameter:
    username
    session
    t                   对应同步内容 ID
    choices             JSON 格式字符串数组，投票选项列表
Action: 为某个同步内容创建对应投票
(1) 成功
    response 200
注：需要管理员权限。
```
---
```
URL: /vote_cheer
Domain: api.dev.hrmes.tv/v1/votes
Method: POST
Parameter:
    username
    session
    t                   投票对应同步内容 ID
    choice              投票选项
    value               尖叫分贝数，范围在 (0, 120]
Action: 进行投票
(1) 成功，返回一个字典，表示当前投票结果
    response 200
    data:
        {
            'a': 23,
            'b': 912,
            'c': 871
        }
```
---
```
URL: /vote_cheer_result
Domain: api.dev.hrmes.tv/v1/votes
Method: GET
Parameter:
    t                   投票对应同步内容 ID
Action: 获取某个投票的结果
(1) 成功
    response 200
    返回结果与 /v1/vote 相同
```


管理员API
===========================================
```
URL: /register_admin (仅用于debug和测试)
Domain: api.dev.hrmes.tv/v1/maintains
Method: POST
Parameters:
        username        String
        password        String
Action: 创建一个新的管理员用户，用户名密码由query发起者指定
(1) 成功
    response: 200
```
---
```
URL: /new_vote_cheer
Domain: api.dev.hrmes.tv/v1/maintains
Method: POST
Parameter:
    username
    session
    t                   对应同步内容 ID
    choices             JSON 格式字符串数组，投票选项列表
Action: 为某个同步内容创建对应投票
(1) 成功
    response 200
注：需要管理员权限。
```
---
```
URL: /new_vote
Domain: api.dev.hrmes.tv/v1/maintains
Method: POST
Parameter:
    username
    session
    t                   对应同步内容 ID
    choices             JSON 格式字符串数组，投票选项列表
Action: 为某个同步内容创建对应投票
(1) 成功
    response 200
注：需要管理员权限。
```
---
```
URL: /new_program
Domain: api.dev.hrmes.tv/v1/maintains
Method: POST
Parameters:
    username            管理员帐号用户名
    password
    name                节目名称
    parent              可选。为整个节目创建 ID 时为空，为某一期节目(episode) 创建 ID 时为整个节目的 ID
Action: 创建新的节目，需要管理员账户和密码
(1) 成功
    response: 200
    properties:
        p:              节目 ID
注：需要管理员权限。
```
---
```
URL: /new_info_thread
Domain: api.dev.hrmes.tv/v1/maintains
Method: POST
Parameters:
    username            管理员用户名
    session
    p                   节目 ID
    name                新建同步内容名称，用于生成该内容下用户间回复产生的消息
Action: 创建新的同步内容主题。
(1) 节目不存在
    response: 403
(2) 成功，返回同步内容 id
    response: 200
    data example:
        {'t': "xxx"}
注：需要管理员权限。
```
