# 如何设计一个在线观看人数功能

> 前言

由于bilibili一直都有稿件在线观看人数的功能，想了解他是如何做到的，因此有了这篇笔记。

## 1、架构选型

### 常用的技术架构

1. 通过Url对access_log等日志记录进行筛选，通过日期分割等操作实现。
    - 缺点
        - 实时性不高
        - IO要求高
    - 优点
        - 实现容易，只需要文本读取
2. WebSocket+Redis
    - 缺点
        - 要求服务器性能
        - 实现稍微复杂
    - 优点
        - 实时性高

### Bilibili采用的方案(猜测)

1. 通过对`network`进行查看，可以看到Bilibili采用的方案是第二种，使用`WebSocket`的方式

2. Bilibili 采用 连接 `wss://broadcast.chat.bilibili.com:7826/sub` 服务。

3. 根据别人的开源来看，应该是传输了这么一段东西，以下称为[data1]

   ```
   0x00, 0x00, 0x00, 0x5B, 0x00, 0x12, 0x00, 0x01, 0x00, 0x00, 0x00, 0x07,
   0x00, 0x00, 0x00, 0x01, 0x00, 0x00, 0x7B, 0x22, 0x72, 0x6F, 0x6F, 0x6D,
   0x5F, 0x69, 0x64, 0x22, 0x3A, 0x22, 0x76, 0x69, 0x64, 0x65, 0x6F, 0x3A,
   0x2F, 0x2F, 0x35, 0x30, 0x33, 0x33, 0x34, 0x35, 0x38, 0x30, 0x2F, 0x38,
   0x38, 0x31, 0x32, 0x30, 0x37, 0x39, 0x32, 0x22, 0x2C, 0x22, 0x70, 0x6C,
   0x61, 0x74, 0x66, 0x6F, 0x72, 0x6D, 0x22, 0x3A, 0x22, 0x77, 0x65, 0x62,
   0x22, 0x2C, 0x22, 0x61, 0x63, 0x63, 0x65, 0x70, 0x74, 0x73, 0x22, 0x3A,
   0x5B, 0x31, 0x30, 0x30, 0x30, 0x5D, 0x7D
   ```

   看起来是16进制的byte数组，解析一下

   ```java
       /**
        * 可以分析出
        *   0x00, 0x00, 0x00, 0x5B, 0x00, 0x12, 0x00, 0x01, 0x00, 0x00, 0x00, 0x07,0x00, 0x00, 0x00, 0x01, 0x00, 0x00
        *   这一部分应该是定义的消息头，无法解析出实际的意义
        *  剩下的部分，解析后是
        *  {"room_id":"video://50334580/88120792","platform":"web","accepts":[1000]}
        */
       @Test
       public void test3() {
           byte[] bytes = {
                  0x7B, 0x22, 0x72, 0x6F, 0x6F, 0x6D,
                   0x5F, 0x69, 0x64, 0x22, 0x3A, 0x22, 0x76, 0x69, 0x64, 0x65, 0x6F, 0x3A,
                   0x2F, 0x2F, 0x35, 0x30, 0x33, 0x33, 0x34, 0x35, 0x38, 0x30, 0x2F, 0x38,
                   0x38, 0x31, 0x32, 0x30, 0x37, 0x39, 0x32, 0x22, 0x2C, 0x22, 0x70, 0x6C,
                   0x61, 0x74, 0x66, 0x6F, 0x72, 0x6D, 0x22, 0x3A, 0x22, 0x77, 0x65, 0x62,
                   0x22, 0x2C, 0x22, 0x61, 0x63, 0x63, 0x65, 0x70, 0x74, 0x73, 0x22, 0x3A,
                   0x5B, 0x31, 0x30, 0x30, 0x30, 0x5D, 0x7D
           };
           StringBuilder result = new StringBuilder();
           for (int index = 0, len = bytes.length; index <= len - 1; index += 1) {
               int char1 = ((bytes[index] >> 4) & 0xF);
               char chara1 = Character.forDigit(char1, 16);
               int char2 = ((bytes[index]) & 0xF);
               char chara2 = Character.forDigit(char2, 16);
               result.append(chara1);
               result.append(chara2);
           }
           System.out.println(new String(new BigInteger(result.toString(), 16).toByteArray()));
       }
   ```

    4. 重点应该在room_id，这里`video`分为两部分，`50334580`应该是视频对应的`av`号（现在使用了bv号，但是实际上还是有对应的av号的），`88120792`这里应该是视频的弹幕信息`cid`

    5. 根据代码来看，应该是只有一个连接地址

       `aid/cid` 作为了一个key，点击视频连接后，进行websocket连接，然后发送[data1],然后服务器返回

       `b'\x00\x00\x00+\x00\x12\x00\x01\x00\x00\x00\x08\x00\x00\x00\x01\x00\x00{"code":0,"message":"ok"}'`

    6. 之后会发送一个`base64`编码过的用户信息之类的？这里我没看懂代码，然后就返回了

       `b'\x00\x00\x00n\x00\x12\x00\x01\x00\x00\x00\x03\x00\x00\x00\t\x00\x00{"code":0,"message":"0","data":{"room":{"online":3,"room_id":"video://85919470/146861497"}}}'`



## 2、我的方案-Java

1. 采用Webscoket连接，用户订阅`/video/{vid}`
2. 根据`vid`创建对应的`Map`存储，`vid`和对应的`websocket-session`
3. 发送给后台消息，带有视频的`userId`
4. 根据`vid`发送到Redis，利用视频的`vid`作为key，使用Redis的`set`数据结构,存储`userId`，通过`scard key` 查看在线人数
5. 通过heatbeat刷新，或者定时刷新。
6. 用户关闭标签，断开websocket服务，带上`vid`告知，移除Redis中的对应的`vid`作为key的`userId`

## 3、不足

由于可能存在用户同时打开多个同一个视频的情况，这时如果关闭，其中一个，则其他也会断开连接，无法时时推送最新的在线观看人数。

> 参考的代码 https://github.com/penpen456/bilibili_online

```python
import websocket
import base64
import requests
import json
import datetime
import time

def get_cid(av):
    response = requests.get("https://api.bilibili.com/x/web-interface/view?aid=" + str(av))
    response.encoding = 'utf-8'
    res = response.text
    # print(res)
    data = json.loads(res)
    c = data['data']['cid']
    # print(c)
    return c

def make_send(av):
    cid = str(get_cid(av))
    res = b'\x00\x00\x00\\\x00\x12\x00\x01\x00\x00\x00\x07\x00\x00\x00\x01\x00\x00{"room_id":"video://' + str(av).encode('utf-8') + '/'.encode('utf-8') + cid.encode('utf-8') + '","platform":"web","accepts":[1000]}'.encode('utf-8')
    return res

def get_online(text):
    cache = text.find(b'"online":')
    # print(cache)
    cache2 = text[cache+9:].find(b',')
    get = int(text[cache+9:cache2+9+cache])
    print(get)
    return get


def connect(plz):
    url = "wss://broadcast.chat.bilibili.com:7823/sub"
    normal = base64.b64decode('AAAAIQASAAEAAAACAAAACQAAW29iamVjdCBPYmplY3Rd') 
    ws = websocket.create_connection(url,timeout=10)
    ws.send(bytes(plz))
    get = ws.recv()
    print(get)
    print(normal)
    ws.send(bytes(normal))
    get = ws.recv()
    print(get)
    if get.find(b'online') != -1:
        # online = get_online(get)
        online=get_online(get)
        return online
    else :
        print("None")

def get_online_from_av(av):
    send = make_send(av)

    online = connect(send)

    return online


def write_file(onlines,times):
    
    with open(file_name,'a') as file_obj:
      file_obj.write(str(times) + ',' + str(onlines) + '\r')



get_online_from_av(85919470)

# file_name = str(input("File_Name(a.txt):"))
# avid = int(input('AVid(85919470):'))




# while True:
    
#     online=get_online_from_av(avid)
#     now_time = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
#     print(now_time)
#     # write_file(online,now_time)

    
#     time.sleep(60)

```

