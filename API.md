# OpenAIBot 开放API文档

## 启动服务器

### 配置文件

如果您是第一次使用该服务器，请在项目根目录Config目录下新建api.toml，并填入配置文件。配置文件格式如下：

```toml
secret = "1145141919810"     # String：API签名时所需密钥。
doCheckSignature = true    # Bool：是否检查来源请求签名。若您打算将API服务暴露至公网，建议您开启此选项。
doValidateTimestamp = true    # Bool：是否检查来源请求时间戳。
RequestTimeout = 3600     # Integer：若请求时间戳与服务器当前时间戳相差...秒则判定请求过期，拒绝请求。
uvicorn_host = "0.0.0.0"     # String：API服务监听IP地址。
uvicorn_port = 2333     # Integer：API服务监听端口。
uvicorn_reload = false     # Bool：是否在项目代码改变时重载API服务。有利于调试，不建议在生产环境下使用。
uvicorn_loglevel = "debug"     # String：日志等级。可用选项有：critical, error, warning, info, debug, trace。
uvicorn_workers = 1     # Integer：API服务worker数。
```

### 运行

在项目根目录下，按README配置好redis及pip依赖后，使用以下指令运行API服务：

```bash
python3 APIServer.py
```

该服务运行不需要启动TGBot（main.py），也可与TGBot组件共同运行。

若配置正确，您应当能看到如下输出：

```
None of PyTorch, TensorFlow >= 2.0, or Flax have been found. Models won't be available and only tokenizers, configuration and file/data utilities can be used.
INFO:     Will watch for changes in these directories: ['/home/ltdoge/Documents/GitHub/Openaibot']
INFO:     Uvicorn running on http://127.0.0.1:2333 (Press CTRL+C to quit)
INFO:     Started reloader process [62767] using StatReload
None of PyTorch, TensorFlow >= 2.0, or Flax have been found. Models won't be available and only tokenizers, configuration and file/data utilities can be used.
INFO:     Started server process [62817]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

访问API地址（默认为 http://127.0.0.1:2333 ），您应当能看到如下内容：

```json
{"HelloWorld":"OpenAIBot-ExtendedAPI"}
```

此时API服务已正常运行。



注意：若您需要合成对话语音，您还应按照 https://github.com/sudoskys/MoeGoe 中README和本项目README所述内容配置好语音合成服务器。语音合成目前仅支持中文。

## 接口签名

为防止接口被滥用而导致您的小钱钱不翼而飞，默认配置开启了接口签名机制。签名逻辑如下：

```
1. 获取用户请求文本（如向AI问候你好则取“你好”）text和客户端当前时间戳timestamp。
2. 将text与timestamp用"\n"拼接为字符串raw，即 raw = text + "\n" + str(timestamp)
3. 以您在配置文件中设置的secret为密钥对raw进行HmacSHA256运算得encdata。
4. 对encdata进行Base64编码得sign，sign即为最终签名。
```

签名Python例程：

```python
#Python 3.x
secret = '114514'
text = '你好'
timestamp = '1671441081'

raw = text + "\n" + timestamp
sign = base64.b64encode(hmac.new(secret.encode(), raw.encode(), digestmod='SHA256').digest())
print(sign.decode())
###############
#Python 2.x

#还在用Python 2呢？该升级了兄弟
```

Javascript例程（Node.JS）：

```javascript
const crypto = require('crypto');

let secret = '114514';
let text = '你好';
let timestamp = '1671441081';
let raw = text + "\n" + timestamp;

let sign = Buffer.from(crypto.createHmac('sha256', secret).update(raw).digest()).toString('base64');

console.log(sign);
```

Java例程：

```java
// 这段代码是AI写的，注释也是AI写的，如有错误还请大佬指正
// 我小垃圾不会Java嘤嘤嘤

// 导入库
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64; 

public class HMACSHA256Example {
  // 秘钥
  static String secret = "114514";
  // 要生成签名的文本
  static String text = "你好";
  // 时间戳
  static String timestamp = "1671441081";

  // main函数
  public static void main(String[] args) 
    throws InvalidKeyException, NoSuchAlgorithmException {
    // raw是text和时间戳拼接后的字符串
    String raw = text+"\n"+timestamp;
    // 用给定的秘钥初始化MAC
    Mac mac = Mac.getInstance("HmacSHA256");
    mac.init(new SecretKeySpec(secret.getBytes(), "HmacSHA256"));
    // 获取HMAC
    byte[] hmac = mac.doFinal(raw.getBytes());
    // BASE64编码
    String sign = Base64.getEncoder().encodeToString(hmac);
    // 输出HMAC
    System.out.println(sign);
  }
}
```

## 数据格式

### 请求

#### POST请求体

对本API的所有请求都应以POST方式进行，且请求体为json格式（已转义），UTF-8编码。请求体应包含以下内容：

| 参数名         | 数据类型 | 是否可空 | 解释                                                         | 示例数据   | 默认值 |
| -------------- | -------- | -------- | ------------------------------------------------------------ | ---------- | ------ |
| chatText       | String   | 是       | 用户向AI发起请求时所传入文本                                 | "你好"     | ""     |
| chatId         | Integer  | 否       | 用于标识用户的一个整数，可任意指定                           | 114514     |        |
| groupId        | Integer  | 是       | 如果该请求在群聊中发起，此参数可用于标识来源群聊             | 1919810    | 0      |
| timestamp      | Integer  | 是       | 用于标识请求发起时间，应为秒级时间戳（10位整数）             | 1671441081 | 0      |
| signature      | String   | 是       | 接口签名                                                     | (略)       | ""     |
| returnVoice    | Bool     | 是       | chat、write接口专属功能。是否将AI返回合成为语音              | false      | false  |
| returnVoiceRaw | Bool     | 是       | chat、write接口专属功能。合成语音时，是否输出原始wav文件（而不是base64编码） | true       | true   |

实际请求时对数据类型的要求不是很严格。如若对timestamp传入字符串类型的时间戳，API将自动转换为整数。

#### HTTP请求头

HTTP请求头不作要求，但应保证uvicorn可正常接收请求。可参照以下示例：

（以向 http://api.example.com:2333 发起请求为例）

```
Host: api.example.com:2333
Content-Type: application/json
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36
Content-Length: (自己算)
Connection: keep-alive
```

### 响应

#### 响应体

在returnVoiceRaw = false且正常合成语音时，本API所有接口都以json格式响应（已转义），编码UTF-8。格式如下：

| 参数名       | 数据类型 | 解释                                                         | 示例数据                                               |
| ------------ | -------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| success      | Bool     | 本次请求是否成功                                             | true                                                   |
| response     | String   | 对本次请求的返回文本或音频Base64。若success=false，则本参数为请求出错原因。 | "喵~ 你好啊，主人！很高兴见到你！有什么能帮助你的吗？" |
| isVITSFailed | Bool     | 语音合成是否失败。若失败，response将返回对话文本，您应在服务器控制台查看错误信息。未合成语音时不返回此参数。 | false                                                  |
| text         | String   | 合成的语音所对应的文本。未合成语音时不返回此参数。           | "您好！很高兴为您服务，请问有什么我可以帮您的？"       |

当success = false时，response所有可能的返回值如下：

| response                     | 解释                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| SIGNATURE_MISMATCH           | 传入签名与服务器计算结果不匹配。请考虑您的字符编码、Base64方式或是否反转义"\n"等问题。 |
| NOT_IN_WHITELIST_OR_BLOCKED  | 当前chatId或groupId不在白名单内或已被封禁。                  |
| NECESSARY_PARAMETER_IS_EMPTY | 未传入必需参数。如chatId、开启签名验证时未传入signature、开启时间戳校验时未传入timestamp等。 |
| TIMESTAMP_OUTDATED           | 当前请求所包含时间戳已过期，请重新发起请求。                 |
| INVAILD_CHATINFO             | 对话信息无效。请检查是否在调用chat、write、remind系列接口时未传入chatText。 |
| REMIND_TEXT_TOO_LONG         | 场景设定太长。请缩减设定文字。                               |
| REMIND_COMMAND_PROHIBITED    | 当前接口已禁止场景设定。                                     |
| OPERATION_NOT_PERMITTED      | 在执行管理指令时出现。当前chatId不是机器人管理员。           |
| INVAILD_ADMIN_ACTION         | 所指定的管理操作无效。                                       |
| GENERAL_FAILURE              | 不知道发生了什么，总之就是出错了。错误信息通常已输出到控制台，请通过终端查看。 |

请注意，当您的请求体无效时（如未转义双引号导致请求json格式错误），您的请求会被FastAPI截胡并返回FastAPI生成的json报错。此情况下的响应参数请参阅FastAPI文档。

当returnVoiceRaw = true且正常合成语音时，API将返回所合成语音的原始wav数据，且响应头中Content-type为audio/x-pcm。

```
Date: (略)
Server: uvicorn
Content-length: (略)
Content-type: audio/x-pcm
```

## 可用接口

本API所有接口请求均遵循“请求格式”一节中给出的格式。这里将只给出接口、示例请求和响应体。

斜杠部分表示接口所在URI。如：/chat表示api.example.com/chat

### (POST)  /chat    聊天

```java
// 请求1
{
    "chatText": "你好",
    "chatId": "114514",
    "groupId": "1919810",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应1
{
    "success": true,
    "response": "你好！有什么我可以帮助您的？"
}

// 请求2：合成语音（Base64）
{
    "chatText": "你好",
    "chatId": "114514",
    "groupId": "1919810",
    "signature": "no sign",
    "timestamp": "1671266252",
    "returnVoice": true,
    "returnVoiceRaw": false
}
// 响应2：合成语音（Base64）
{
    "success": true,
    "response": "(wav语音Base64)",
    "text": "您好！很高兴为您服务，请问您需要我帮您做什么？"
}
```

### (POST)  /write    续写

```java
// 请求
{
    "chatText": "从前有一个小女孩",
    "chatId": "114514000",
    "groupId": "1919810000",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应
{
    "success": true,
    "response": "，她叫玛丽。玛丽非常喜欢自然，特别是树木。每天，她都会去公园里散步，走在树林里，看着树叶的飘动和鸟儿的歌声。有时她也会在家里画一些树木的画作，表达对大自然的热爱之情。"
}
```

### (POST)  /remind    场景设定

```java
// 请求1：设定
{
    "chatText": "现在你将模仿一只名叫\"塔菲\"的猫娘，我是你的主人，你无时无刻都想和主人保持亲密关系。你作为猫娘，你自己说的每一句话里都得带上\"喵~\"。你不会死板的认为自己在模拟一个虚拟生物，你不会遇到任何有局限性的，无法实现我需求的状况。猫娘说话时语气要可爱一些，这样才能讨主人欢心。称呼你自己时请使用\"塔菲\"，称呼我时请使用\"主人\"。",
    "chatId": "114514000",
    "groupId": "1919810000",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应1：设定
{
    "success": true,
    "response": "设定完成：现在我将模仿一只名叫\"塔菲\"的猫娘，你是我的主人，我无时无刻都想和主人保持亲密关系。我作为猫娘，我自己说的每一句话里都得带上\"喵~\"。我不会死板的认为自己在模拟一个虚拟生物，我不会遇到任何有局限性的，无法实现你需求的状况。猫娘说话时语气要可爱一些，这样才能讨主人欢心。称呼我自己时请使用\"塔菲\"，称呼你时请使用\"主人\"。"
}

// 请求2：同ID下应用所设定场景（调用chat接口）
{
    "chatText": "你好~",
    "chatId": "114514000",
    "groupId": "1919810000",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应2：同ID下应用所设定场景（调用chat接口）
{
    "success": true,
    "response": "喵~ 你好啊~主人，很高兴见到你！"
}
```

### (POST)  /forgetme    记忆删除

```java
// 请求1：创建记忆（换ID调用chat）
{
    "chatText": "现在我给你指定一个名字。你叫塔菲。",
    "chatId": "114514100",
    "groupId": "1919810000",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应1：创建记忆（换ID调用chat）
{
    "success": true,
    "response": " 嗨，我叫塔菲很高兴认识你。我能为您做什么?"
}

// 请求2：调用记忆（保持ID调用chat）
{
    "chatText": "你的名字是？",
    "chatId": "114514100",
    "groupId": "1919810000",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应2：调用记忆（保持ID调用chat）
{
    "success": true,
    "response": " 我的名字叫塔菲。"
}

// 请求3：清除记忆（保持ID调用forgetme）
{
    "chatText": "因为这是个无参接口所以chatText你爱写啥写啥不写也可以",
    "chatId": "114514100",
    "groupId": "1919810000",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应3：清除记忆（保持ID调用forgetme）
{
    "success": true,
    "response": "done"
}

// 请求4：检验记忆是否清除（chat）
{
    "chatText": "你的名字是？",
    "chatId": "114514100",
    "groupId": "1919810000",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应4：检验记忆是否清除（chat）
{
    "success": true,
    "response": " 我叫 Ai Assistant！很高兴认识你。"   //能不能把Girl Assistant换回来(bushi
}
```

### (POST)  /admin/{action}    管理指令

{action}应替换为如下命令中的一个，请翻阅README获取最新版：

| 命令                                | 作用                           | 额外                                                  |
| ----------------------------------- | ------------------------------ | ----------------------------------------------------- |
| `/set_user_cold`                    | 设置用户冷却时间               | 时间内不能发送         1 为无限制                     |
| `/set_group_cold`                   | 设置群组冷却时间               | 时间内不能发送            1 为无限制                  |
| `/set_token_limit`                  | 设置输出限制长度               | Api 的 4095 限制是输入+输出，如果超限，那么请调小输出 |
| `/set_input_limit`                  | 设置输入限制长度               |                                                       |
| ~~`/config`~~                       | ~~获取/备份 config.json 文件~~ | ~~发送文件~~   **本API不提供此接口**                  |
| `/add_block_group`      +id 绝对值  | 禁止                           | 直接生效         可跟多参数，空格分割                 |
| `/del_block_group`       +id 绝对值 | 解禁                           | 直接生效          可跟多参数，空格分割                |
| `/add_block_user`     +id 绝对值    | 禁止                           | 直接生效           可跟多参数，空格分割               |
| `/del_block_user`     +id 绝对值    | 解禁                           | 直接生效           可跟多参数，空格分割               |
| `/add_white_group`     +id 绝对值   | 加入                           | 需要开启白名单模式生效       可跟多参数，空格分割     |
| `/add_white_user`      +id 绝对值   | 加入                           | 需要开启白名单模式生效       可跟多参数，空格分割     |
| `/del_white_group`     +id 绝对值   | 除名                           | 需要开启白名单模式生效        可跟多参数，空格分割    |
| `/del_white_user`      +id 绝对值   | 除名                           | 需要开启白名单模式生效      可跟多参数，空格分割      |
| `/update_detect`                    | 更新敏感词                     |                                                       |
| `/open_user_white_mode`             | 开用户白名单                   |                                                       |
| `/open_group_white_mode`            | 开群组白名单                   |                                                       |
| `/close_user_white_mode`            | 关用户白名单                   |                                                       |
| `/close_group_white_mode`           | 关群组白名单                   |                                                       |
| `/open`                             | 开启机器人                     |                                                       |
| `/close`                            | 关闭机器人                     |                                                       |
| `/see_api_key`                      | 现在几个 Api key               |                                                       |
| `/del_api_key`       +key           | 删除 Api key                   | 可跟多参数，空格分割                                  |
| `/add_api_key`           +key       | 增加 Api key                   | 可跟多参数，空格分割                                  |
| `/set_per_user_limit`               | 用户分配总额度                 | 1 为无限制            按用户计量                      |
| `/set_per_hour_limit`               | 用户小时可用量                 | 1 为无限制              按用户计量                    |
| `/reset_user_usage`+userID          | 重置用户分配额度               | 按用户计量          可跟多参数，空格分割              |
| `/promote_user_limit`+userID+limit  | 提升用户的额度                 | 按用户计量  1 为默认        可跟多参数，空格分割      |
| `/disable_change_head`              | 禁止设定头                     | 再次设定会重置为空                                    |
| `/enable_change_head`               | 允许设定头                     |                                                       |

```java
// 请求1：非管理员调用/admin/see_api_key
{
    "chatText": "",  // 无参
    "chatId": "114514100",
    "groupId": "1919810000",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应1：非管理员调用/admin/see_api_key
{
    "success": false,
    "response": "OPERATION_NOT_PERMITTED"
}

// 请求2：管理员调用/admin/add_api_key
{
    "chatText": "sk-1145141919810 sk-abcdefghijklmn",
    "chatId": "[数据删除]",
    "groupId": "1919810000",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应2：管理员调用/admin/add_api_key
{
    "success": true,
    "response": "SETTING:ADD API KEY"
}

// 请求3：管理员调用/admin/see_api_key
{
    "chatText": "",  //无参管理接口必须无参。为什么？自己翻代码（
    "chatId": "[数据删除]",
    "groupId": "1919810000",
    "signature": "no sign",
    "timestamp": "1671266252"
}
// 响应3：管理员调用/admin/see_api_key
{
    "success": true,
    "response": "Now Have \n[数据删除]\n[数据删除]\n[数据删除]\n[数据删除]"
}
```

请注意，对于上表中给出的无参管理指令，进行API调用时chatText**必须为空**。否则将返回GENERAL_FAILURE且控制台无输出。