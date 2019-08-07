qingteng_love

青藤之恋小程序 破解 批量获取数据，并分析

#### score
青藤之恋是一款最近比较火的小程序，标签，高学历交友。学历都真实可查，并且资料的审核很严格。我个人还是很喜欢这款小程序的，还推荐过给身边的朋友使用，别人还在上面找到了另一半，很神奇。
我对青藤之恋这个小程序比较看好的另一个原因是因为，这是我的老东家，微派做的产品。虽然我在微派的时间不长，但是整个公司还是给我留下了非常深刻的印象。团队的综合素养在武汉的互联网公司还是非常出众的。不论是产品，技术，测试，设计，运营还是运维，都非常优秀，所以这样的团队做出这款产品，还是意料之中的事情。
其实我自己个人一直对相亲交友这块感兴趣，随着现在的人越来越宅，单身狗越来越多，网络相亲越来越成为了一个刚需，这块市场很大。但是目前做得好，用户体验出色的产品基本没有，传统的婚恋网站都是坑蒙拐骗类型的，而且很多用户花钱了并没有什么好的体验，更别提被人骗的经历了，所以这块未来可期。青藤之恋这个产品，适时出现了。
说了这么多，进入主题。
青藤之恋，每天只推荐5个用户，这让人很抓狂，有没有办法，破解一下，多看一些用户呢？带着这个目的，我研究了一下。
最后的结果是，确实只推荐5个，但是，我可以把一定量的用户数据download下来，不过这样只能看看，研究下数据，不能去进行like，私信等操作。有了这个就够了，反正，我也只是为了研究一下相关的数据。 看下青藤之恋上面到底有多少用户，用户的学历，年龄，地域，行业等等。

#### 抓包
第一步，首先就是抓包。
这里，我用的是mitmproxy。相关的介绍和使用请参考其他资料。
抓包的结果：
![](http://ww1.sinaimg.cn/large/007dl3HPgy1g5ozhk8awkj31ec0tgn3u.jpg)

![](http://ww1.sinaimg.cn/large/007dl3HPgy1g5ozhtgu9vj31ec0tggr5.jpg)

![](http://ww1.sinaimg.cn/large/007dl3HPgy1g5ozi0ev79j31ec0tg43i.jpg)

上图显示的是推荐列表这个接口。参数只有一个platform=2,这个应该是iOS平台的，其他的参数都没有了，然后header中还有几个重要的参数需要重点关注。

##### authorization
这个是登录认证，登录成功后，应该都会有这个参数，固定的。

##### appversion
这个参数应该也是有用的，在后面源码中看到了相关的设置

###### user-agent
这个不用说了，后面用python来爬数据肯定是需要的

##### sign
然后最重要的就是这个sign了，是一个加密的签名。
根据我自己的摸索，不同的接口这个参数是不一样的，所以这个加密跟接口的名称是有关系的。当然具体是怎么来的，这个就只能去小程序的源码中一探究竟了。



#### 小程序源码
获取小程序源码，网上也有相关资料，我这里用一台root了的Android手机，也拿到了小程序的wapkg文件，反编译后得到了一份不算完整的源码。
![](http://ww1.sinaimg.cn/large/007dl3HPgy1g5ozrfzgjqj31wy176tku.jpg)
然后就可以在这里面找sign的具体实现了。
直接把报错部分的代码注释掉，保留没问题的代码，然后在代码中断点，微信开发工具上面直接编译，就可以进行debug操作了。
![](http://ww1.sinaimg.cn/large/007dl3HPgy1g5p018jjp1j317e0icjv2.jpg)
因为小程序刚启动的时候就会调用api接口，所以，一开始就可以debug到接口加密的地方，发现了Sign生成的地方。
最后找到了这段代码，这个地方就是获取签名的过程，用的是SHA-1算法，用的是jssha这个第三方库来进行加密的。
关于这个库，有个网站可以直接测试加密结果，很方便，调试的时候可以测试。
> https://caligatio.github.io/jsSHA/
![](http://ww1.sinaimg.cn/large/007dl3HPgy1g5ozu0eoeoj30ni0e8wgc.jpg)

最后，根据上面的一顿操作，Sign就可以拿到了，经过跟前面抓包工具得到的数据比对，完全ok。有了sign，然后，就可以干点事了。

#### request请求获取用户信息
构造request请求，获取具体某个用户的信息,并存入mongo中
```python
def get_user_info(sign, target_uid):
    url = host + "/user/get_user_info"
    headers={
        "content-type": "application/x-www-form-urlencoded",
        "AppVersion": "2.2.3",
        "Sign":sign,
        "authorization": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJVaWQiOjk2MjUxLCJleHAiOjE1NjYzNjA1MDh9.LW4dT8fqs7zXNLkB_YYqYDIqeTRwb0v3-vcjjZ74keY",
        "user-agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 12_3_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 MicroMessenger/7.0.5(0x17000523) NetType/WIFI Language/zh_CN"
    }

    params = {
        "target_uid": target_uid,
        "platform": 2
    }
    try:
        response = requests.post(url,data=params,headers= headers,stream=False,timeout=20)
        user_json = json.loads(response.text)
        user_json = user_json["data"]
        # print(user_json)
        # uid为0 无效用户 过滤掉
        if user_json["uid"] == 0:
            print('无效用户')
            return
        # 注销用户 过滤掉
        if user_json["nickname"] == "用户已注销":
            print('已注销用户')
            return
        # 学校级别
        school_level = user_json["school_level"]
        if school_level == 0:
            school_level_str = "无"
        if school_level == 1:
            school_level_str = "211及以上"
        elif school_level == 2:
            school_level_str = "一本"
        elif school_level == 3:
            school_level_str = "普通本科"
        elif school_level == 4:
            school_level_str = "专科"

        # 性别
        gender = user_json["gender"]
        if gender == 1:
            gender_str = "男"
        else:
            gender_str = "女"
 
        user = {
            "uid": user_json["uid"],
            "nickname": user_json["nickname"],
            "gender": gender_str,
            "avatar_url": user_json["avatar_url"],
            "birthday": user_json["birthday"],
            "age":user_json["age"],
            "province": user_json["province"],
            "city": user_json["city"],
            "phone": user_json["phone"],
            "school": user_json["school"],
            "school_level": user_json["school_level"],
            "school_level_str": school_level_str,
            "education": user_json["education"],
            "education_type": user_json["education_type"],
            "stature": user_json["stature"],
            "wid": user_json["wid"],
            "about_me": user_json["about_me"],
            "hobbies": user_json["hobbies"],
            "emotional_view": user_json["emotional_view"],
            "about_half": user_json["about_half"],
            "annual_salary": user_json["annual_salary"],
            "is_show": user_json["is_show"],
            "photos": user_json["photos"],
            "check_status": user_json["check_status"],
            "profession": user_json["profession"],
            "inviter": user_json["inviter"],
            "horoscope": user_json["horoscope"],
            "is_vip": user_json["is_vip"],
            "wxid": user_json["wxid"],
            "is_show_vip": user_json["is_show_vip"],
            "photo_list": user_json["photo_list"],
            "is_liked": user_json["is_liked"],
            "is_super_liked": user_json["is_super_liked"],
            "two_way_like": user_json["two_way_like"],
            "moment_photos": user_json["moment_photos"],
            "moment_total": user_json["moment_total"],
            "refuse_reason": user_json["refuse_reason"]
        }
        # condition = {'uid':user["uid"]}   更新
        # db.user_info.update_one(condition, {'$set':user})
        db.user_info.insert_one(user)
        response.close()
        print("mongo保存成功")
    except Exception as e:
        print('Error is----------', e)
```

#### 批量获取
批量获取并存入MongoDB
```python
def get_info_sign(start, end):
    # mongo中取出sign和user_id拼接的字符串 拆开后即可构建header中的sign和param的target_uid
    datas = db.user_info_signs.find({"target_id":{'$gte':start, '$lte':end}})
    # datas = db.user_info_signs.find({"target_id":20000})
    for data in datas:
        # 睡眠1~3秒 开始睡眠是担心被反爬，后来发现并没有，所以取消了睡眠
        # time.sleep(random.randint(1,3))
        # time.sleep(1)
        strs = data["sign"].split(":")
        sign = strs[0]
        target_uid = strs[1]
        print('sign is', sign, 'target_uid is ', target_uid)
        get_user_info(sign, target_uid)
```
#### 用matplotlib绘图
饼图 柱状图等等
