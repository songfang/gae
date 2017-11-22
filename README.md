# GAE-DSP - General Advertising Engine
GAE旨在创建一个**开箱即用**的通用广告投放引擎,业务模型适用于DSP平台,只需要按规定格式灌输索引文件即可直接上线使用,且具备基本的定向功能。
网络通讯层使用Netty,应用层SpringBoot(不包含servlet容器)。

数据传输系统[GAE-DAS](https://github.com/wanghongfei/gae-das),通过监听mysql binlog来自动生成投放引擎所需的增量索引数据; GAE会支持从kafka中读取增量索引

## 关于定向
- 地域定向

GAE通过请求参数中的IP字段实现按地域匹配广告功能,**需要下载ip字典**
- 人群标签定向

GAE支持人群标签定向,但标签的获取需要自己实现(如可以调DMP服务, 已预留出接口)。
标签包括`type`和`id`两个属性, 分别表示标签类型(如年龄性别)和标签id。
GAE并没有规定必须用哪些类型的标, 只负责通过标签进行触发和过虑广告。

## 关于索引
索引分成两类，**全量索引**和**增量索引**. GAE在启动时一定性读取全部全量索引, 在运行期间会监控增量索引并实时加载更新.
全量索引只能从文件中读取，增量索引可以从文件也可以从kafka中读取. 如果从文件中读取增量, 则先会从第0个文件开始读取, 当下一个文件出现时自动切换至新文件。
例如,只有`gae.idx.incr.0`存在时则会一直监控、读取该文件，当`gae.idx.incr.1`出现时会切换从新文件开始读取。

## 功能
![function](http://ovbyjzegm.bkt.clouddn.com/gae-route.png)

## 构建运行
### 下载IP字典
```
wget http://ovbyjzegm.bkt.clouddn.com/ipdict.tar.gz
tar -zxvf ipdict.tar.gz
```
### 运行
```
mvn clean package -Dmaven.test.skip=true
```
GAE的增量索引加载有两种方式，监控索引文件增量或从kafka中读取. 当从文件中加载时, 需在运行时打开kafka开关(kafka连接配置详见`application.yaml`):
```
java -jar target/gae.jar --gae.server.port=9000 --gae.index.kafka=true --gae.index.file.path=./ --gae.index.file.name=gae.idx --gae.dict.ip=IP字典文件名
```
其中`--gae-index.kafka`控制是否从kafka中读取增量索引, `gae.idx`为全量索引文件名. 地域ID与城市名称的对应关系见: `wget http://ovbyjzegm.bkt.clouddn.com/reg.txt`

当从文件中加载增量索引时, 则不需要打开kafka开关(无需指定`--gae.index.kafka=true`), 默认为关闭。

## 项目进度
基本完成:
所有功能ready, 在做最后的测试

## 模块说明 org.fh.gae.*
- net

HTTP网络通讯逻辑, netty启动入口`net.GaeHttpServer`

- query

广告检索逻辑


## 测试
一次可请求多个广告位:
```
curl -X POST \
  http://127.0.0.1:9000/ \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -d '{
    "request_id": "hello",
    "auth": {
        "tid": "tid",
        "token": "token"
    },
    "device": {
        "ip": "61.135.169.78",
        "mac": "AA:BB:CC:DD:EE:FF",
        "id": "IDFA",
        "type": 1
    },
    "slots": [
        {
            "slot_id": "广告位id",
            "slot_type": 1,
            "w": 1920,
            "h": 1080,
            "material_type": [1,2]
        },
        {
            "slot_id": "广告位id2",
            "slot_type": 1,
            "w": 1920,
            "h": 1080,
            "material_type": [1,2]
        }
    ]
}'
```
响应:
```
{
    "code": 0,
    "result": {
        "ads": [
            {
                "ad_code": "adCode1",
                "h": 1080,
                "land_url": "http://www.163.com",
                "material_type": 1,
                "show_urls": [
                    "http://www.gae.com/showMonitor.gif?sid=a",
                    "http://www.gae.com/showMonitor.gif?sid=b"
                ],
                "slot_id": "广告位id",
                "url": "http://www.baidu.com/xxx.jpg",
                "w": 1920
            },
            {
                "ad_code": "adCode1",
                "h": 1080,
                "land_url": "http://www.163.com",
                "material_type": 1,
                "show_urls": [
                    "http://www.gae.com/showMonitor.gif?sid=a",
                    "http://www.gae.com/showMonitor.gif?sid=b"
                ],
                "slot_id": "广告位id2",
                "url": "http://www.baidu.com/xxx.jpg",
                "w": 1920
            }
        ],
        "request_id": "hello"
    }
}
```
可以通过`IndexGenerator.genIndex()`方法随机生成全量索引文件进行测试
