## Logstash
### 工作流  
&emsp; Logstash事件处理管道包括三个阶段：inputs -> filters -> outputs。inputs生成事件，filters修改事件，outputs把他们发送到其它地方。inputs和outputs支持codecs，允许你编码或者解码数据像它进来的时候或者退出管道的时候不需要经过一个额外的过滤器  
&emsp; 常见的inputs有：file、syslog、redis、beats
&emsp; 常见的filter有：grok、mutate、drop、clone、geoip
&emsp; 常见的outputs有：elasticsearch、file、graphite、statsd
&emsp; codecs是input或者output中基本的流过滤器，常见的codecs包括：json、msgpack、plain、multiline  
### 启动
* 校验配置文件：bin/logstash -f first-pipeline.conf --config.test_and_exit
* 启动：bin/logstash -f first-pipeline.conf --config.reload.automatic

### 优缺点
1. 丰富的插件类型可以满足大部分的需求；
2. Logstash向ES写数据，如果ES不可达，可能存在丢失数据的情况；
