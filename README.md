# 当前的问题
在开发钉钉ISV应用的时候，会发现钉钉ISV应用必须部署有公网IP的服务器上，也就是在开发应用之前必须先准备购买阿里云等服务器资源。在之后的开发中，由于ISV应用的回调地址必须是公网域名或IP之下，对于大部分开发者来说，开发者无法在本地调试远程代码，对于回调URL校验不通过之类问题无法追踪，只能不断远程部署来看log日志来调试修改，本文主要讲解如何解决这些问题。
# 解决方案
基于以上问题，我们设计了一套支持ISV应用本地化调试的方案，使用这套方案可以达到以下目的：
1. ISV应用开发阶段不需要购买阿里云等公网机器资源
2. 开发调试应用可以在本地（eclipse+Tomcat）进行，不需要部署到远程公网机器上
# 启动内网穿透
#### 1.下载工具
```
git clone https://github.com/open-dingtalk/pierced.git
```


![image.png | left](https://gw.alipayobjects.com/zos/skylark/public/files/59fd4d93eaf4c0323df64cedaff08fb6.png "")

启动工具，执行命令“./ding -config=./ding.cfg -subdomain=域名前缀 端口”，以mac为例：
```
cd mac_64
chmod 777 ./ding
./ding -config=./ding.cfg -subdomain=abcde 8080
```
启动后界面如下图所示：


![image.png | left](https://gw.alipayobjects.com/zos/skylark/public/files/7552c06dd9a922602d13f56652940377.png "")

参数说明：

| 参数 | 说明 |
| :--- | :--- |
| config | 钉钉提供内网穿透的配置文件，不可修改 |
| subdomain | 你需要使用的域名前缀，该前缀将会匹配到“vaiwan.com”前面，例如你的subdomain是abcde，你启动工具后将会将abcde.vaiwan.com映射到本地 |
| 端口 | 你需要代理的本地服务http-server端口，例如你本地端口为8080等 |

#### 2.启动完客户端后，你访问http://abcde.vaiwan.com/xxxxx都会映射到 [http://127.0.0.1:8080/xxxxx](http://127.0.0.1:8080/xxxxx)
注意：
> 1.你需要访问的域名是http://abcde.vaiwan.com/xxxxx 而不是http://abcde.vaiwan.com:8082/xxxxx
> 2.你启动命令的subdomain参数有可能被别人占用，尽量不要用常用字符，可以用自己公司名的拼音，例如我使用：alibaba、dingding等。
> 3.可以在本地起个http-server服务，放置一个index.html文件，然后访问http://abcde.vaiwan.com/index.html测试一下。
# 准备开发环境
下载eclipse或者intelliJ，Tomcat等搭建本地java项目环境，不一一赘述。
# DEMO部署
#### 下载示例代码
```
git clone https://github.com/open-dingtalk/isv-demo-java-miniapp.git
```
__请开发者使用jdk1.6或以上版本__。
#### __导入数据库文件__
提示：需要提前新建ding\_isv\_access数据库，详见“准备工作”第4步。
```
mysql -u root -p ding_isv_access < db_sql.sql
```
此步骤可能会遇到如下报错：
> COLLATION 'utf8\_general\_ci' is not valid for CHARACTER SET 'utf8mb4'
请将db\_sql.sql以文本形式打开，并将文件中所有“utf8mb4”字符串全部替换为“uft8”。
# 创建小程序并配置代码
登录钉钉开发者平台，创建一个小程序，如下图所示：



![image.png | left | 748x279](https://cdn.yuque.com/lark/0/2018/png/20097/1524195256423-a24d2f7d-5c76-4a36-8542-fe28d0aaa9ee.png "")


请填写应用的名称、应用LOGO、应用描述、Token、数据加密秘钥、应用类型、应用IP白名单等信息并保存。




![image.png | left | 748x619](https://cdn.yuque.com/lark/0/2018/png/20097/1524195373143-fe5133d6-38b6-46e9-af93-74640afe20bc.png "")


选择类型为“测试应用”
__注意：类型一旦选定不能进行修改，测试应用主要是用来做开发调试，不能发布商品、不能生成线下部署二维码。__

创建完成后，我们在小程序的基础信息里能拿到如下参数：
> Token: 随意填写任意字符串
> suiteKey：应用key
> suiteSecret：应用秘钥
> AESKey：数据加密密钥，点击自动生成

在MySQL数据库中插入下面信息，注意替换“应用名称、suiteKey、suiteSecret、EncodingAESKey、Token”中的值：

```
insert into isv_suite(id, gmt_create, gmt_modified, suite_name, suite_key,
suite_secret, encoding_aes_key, token, event_receive_url)
values(1, NOW(), NOW(), '应用名称', 'suiteKey', 'suiteSecret','EncodingAESKey', 'Token', '');
```

我们用IDE打开示例DEMO代码，配置好如上参数。
1.ding-isv-access/web/src/main/webapp/WEB-INF/config.properties 里面配置好system.env、oapi.environment、corp.suite.callback、jdbc.url、db.username、db.password、suite.suiteKey，如下所示：

```
oapi.environment=https://oapi.dingtalk.com
corp.suite.callback=http://xxxxx.vaiwan.com/ding-isv-access/suite/corp_callback/
#jdbc.url=jdbc:mysql://127.0.0.1:3306/ding_isv_access?useUnicode=true&amp;characterEncoding=utf8
jdbc.url=
db.username=
db.password=
suite.suiteKey=
```

| 配置项 | 配置说明 |
| :--- | :--- |
| oapi.environment | 钉钉服务端接口地址 |
| corp.suite.callback | 应用通讯录回调地址，启动内网穿透后，将xxxxx.vaiwan.com修改为自己的subdomain地址。 |
| jdbc.url | 数据库连接池地址 |
| db.username | 数据库用户名 |
| db.password | 数据库密码 |
| suite.suiteKey | 创建应用拿到的suitekey |

2.用IDE打开ding-isv-access/web/src/main/webapp/WEB-INF/log4j.xml文件，配置好LOG\_PATH，也就是日志文件地址。

```
<property name="LOG_PATH" value="/usr/local/logs/ding-isv-access" />
```

3.完成上面的配置后，请将代码部署到本地Tomcat下面，如果配置无误，打开浏览器输入地址：http://abcde.vaiwan.com/ding-isv-access/checkpreload.htm，会看到浏览器中显示success，说明项目已经部署成功。

> 注意：我们提供的内网穿透服务可能会有延迟，如果页面刷新不出来，请稍等3~5分钟后再试。

# 验证回调
在小程序的基础信息里面点击右上角"修改"按钮设置回调URL，在回调URL地址上填写通过远程代理的域名服务器地址，例如：http://abcde.vaiwan.com/ding-isv-access/suite/callback/{suitekey}。

__注意：{suitekey}替换为自己应用的真实suitekey__

点击“验证有效性”验证成功并保存，如下图所示：




![image.png | left | 748x511](https://cdn.yuque.com/lark/0/2018/png/20097/1524195931056-5c51191c-6985-46ef-a120-b12a8a76ec1a.png "")



![image.png | left | 748x683](https://cdn.yuque.com/lark/0/2018/png/20097/1524196128774-2f9175a0-9171-4df9-bb6b-e6bfcd013482.png "")


# 开发小程序
请参考使用[钉钉开发工具开发小程序](https://open-doc.dingtalk.com/microapp/isv/oq1uge)，并发布一个版本。
#### 获取当前企业的corpId
在​app.js里面拿到corpId 存储在globalData里面
```plain
App({
  onLaunch(options) {
    this.globalData.corpId = options.query.corpId;
  },
  onShow() {
    console.log('App Show');
  },
  onHide() {
    console.log('App Hide');
  },
  globalData: {
    corpId: "",
    hasLogin: false,
  },
});
```

#### 获取免登授权码

```plain
var app = getApp();
var me = this;

requestAuthCode({
      corpId: app.globalData.corpId
    }).then((res) => {
      my.httpRequest({
        url: 'http://qwert.vaiwan.com/ding-isv-access/get_user_info',
        // 获取用户信息的服务端POST接口地址
        method: 'POST',
        data: {
          code: res.code,
          corpId:app.globalData.corpId,
          url:"http://qwert.vaiwan.com/ding-isv-access/get_user_info"
        },
        dataType: 'json',
        success: function(res) {
          console.log(res);
          me.setData({
            userName : res.data.userId,
            deviceId : res.data.deviceId
          });
        },
        fail: function(res) {
        },
        complete: function(res) {
          //my.hideLoading();
          //my.alert({content: 'complete'});
        }
        
      });
        console.log(`auth success, authCode is ${res.code}`);
    }).catch((err) => {
        console.log(`auth fail, error message is ${JSON.stringify(err)}`);
    })
```
# 发布版本
请参考小程序开发者工具IDE的使用[发布一个版本](https://open-doc.dingtalk.com/microapp/isv/oq1uge#%E5%8F%91%E5%B8%83)。
# 预览小程序
请参考[钉钉小程序发布预览](https://open-doc.dingtalk.com/microapp/isv/gwads2)。
