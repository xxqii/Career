## LogStash Filter
### age
 * 安装：bin/logstash-plugin install logstash-filter-age  
 * 简介：计算事件的年龄。当前时间戳减去Event的时间戳，通过此filter可以丢弃年龄比较大的时间。例如：  
 * 文档：https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-age.html#_getting_help_55
 > filter {  
   &emsp; age {}  
   &emsp; if [@metadata][age] > 86400 {  //  过滤一天前的事件  
   &emsp; &emsp; drop {}  
   &emsp; }  
 > }
### aggregate
 * 安装： bin/logstash-plugin install logstash-filter-aggregate  
 * 简介：相同任务的不同日志中聚合信息，将最终的聚合信息存放到最终事件中   
 * 文档：https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-aggregate.html  
### alter
 * 安装：bin/logstash-plugin install logstash-filter-alter  
 * 简介：修改不包含在mutate过滤器中的字段  
 * 注意：在后面的版本中会合并在mutate过滤器中  
### anonymize
 * 安装：-
 * 简介：采用一致性hash值替换原来字段的值  
 * 文档：https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-anonymize.html  
 > anonymize {  
   &emsp;algorithm => ...   
   &emsp;fields => ...  
   &emsp;key => ...  
 > }  
 
 *algorithm必须为["SHA1", "SHA256", "SHA384", "SHA512", "MD5", "MURMUR3", "IPV4_NETWORK"]中的一个*  
### cidr
 * 安装：-
 * 简介：CIDR过滤器用于根据可能包含IP地址的网络块列表检查事件中的IP地址。可以针对多个网络检查多个地址，任何匹配都会成功。成功后，可以在事件中添加其他标记和/或字段。  
 * 文档：https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-cidr.html  
### cipher
 * 安装：-  
 * 简介：此过滤器解析数据源，并在将其存储到目标之前应用加密或解密程序。
 * 文档：https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-cipher.html   
### clone
 * 安装：-  
 * 简介：CIDR过滤器用于根据可能包含IP地址的网络块列表检查事件中的IP地址。可以针对多个网络检查多个地址，任何匹配都会成功。成功后，可以在事件中添加其他标记和/或字段。  
 * 文档：https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-clone.html  
### collate
 * 安装：bin/logstash-plugin install logstash-filter-collate  
 * 简介：通过日志时间合并不同数据源的事件  
 * 文档：https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-collate.html#plugins-filters-collate  

> filter {  
  &emsp;collate {  
  &emsp;&emsp;size => 3000  
  &emsp;&emsp;interval => "30s"  
  &emsp;&emsp;order => "ascending"  
  &emsp;}  
  }   

*日志每3000条或30s校对一次，order必须为["ascending", "descending"]中的一个*  
### csv
 * 安装：-
 * 简介：从Event的csv字段中获取数据，解析并存储到独立的字段中（可以解析任何分隔符的数据，不仅限于','）  
 * 文档： https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-csv.html  
### date
 * 安装：-
 * 简介：解析Event中的对应字段，并使用解析结果（date或者timestamp）作为这个Event的timestamp字段使用  
 * 文档：https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-date.html
### de_dot
 * 安装：-  
 * 简介：替换字段中的','为其它符号，需要拷贝源字段到新字段，然后删除原字段。（操作消耗性能，不推荐使用）  
 * 文档：https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-date.html  
### dissect
 * 安装：bin/logstash-plugin install logstash-filter-dissect  
 * 简介：一种分隔字段的操作  
 * 文档：https://www.elastic.co/guide/en/logstash/5.3/plugins-filters-dissect.html  
### dns
 * 简介：在reverse指定的字段上执行lookup命令，或者逐个的在resolve上指定的字段执行lookup  
*一次执行一个Event，会明显降低管道的吞吐量*  
### drop
 * 简介：丢弃Event
### elapsed
 * 安装：bin/logstash-plugin install logstash-filter-elapsed  
 * 简介：跟踪开始和结束Event，计算之间消耗的时间（使用Event中的timestamp字段计算时间差）  
### elasticsearch
 * 安装： bin/logstash-plugin install logstash-filter-elasticsearch  
 * 简介：从ES中查询前一个Event的内容，拷贝一些字段到当前Event  
### emojo
 * 安装：bin/logstash-plugin install logstash-filter-emoji  
 * 简介：将字段中的名字或者编码转化为emojo表情  
### environment
 * 安装：bin/logstash-plugin install logstash-filter-environment  
 * 简介：将环境变量添加到@metadata字段下  
### extractnumbers
 * 安装：bin/logstash-plugin install logstash-filter-extractnumbers  
 * 简介：提取字符串中的数字
### fingerprint
 * 安装： -  
 * 简介：一致性hash替换值
### geoip
 * 安装： -  
 * 简介： 查找geo地址信息
### grok
 * 简介：解析任意文本并结构化
### i18n
 * 简介：移除字段中的特殊字符
### jdbc_streaming
 * 安装： bin/logstash-plugin install logstash-filter-jdbc_streaming  
 * 简介：查找数据库中的字段，并将结果合并到当前文档的target字段中
### json
 * 简介：解析json格式的字符串字段，并将结果合并到当前Event中  
### json_encoding
 * 安装：bin/logstash-plugin install logstash-filter-json_encode  
 * 简介：将字段序列化为json  
### kv
 * 简介：解析k-v格式的字符串
### metaevent
 * 安装： bin/logstash-plugin install logstash-filter-metaeven  
### metricize
 * 安装： bin/logstash-plugin install logstash-filter-metricize  
 * 简介：如果一个Event字段中包含多个metricize，则将字段分隔并封装为多条Event  
### metrics
 * 简介：对统计规律非常有益处  
### mutate
 * 简介：字段基因突变，修改名字，删除，替换和修改字段
### oui
 * 简介：从mac地址中解析oui数据
### prune
 * 简介：从Event中移除白名单、黑名单中的字段或者值
### punct
### range
 * 安装：bin/logstash-plugin install logstash-filter-range  
 * 简介：检查特定字段的大小、长度。
### ruby
 * 简介：执行ruby代码
### sleep
 * 简介：Sleep一段时间，对频率限制的时候使用  
### split
 * 简介：切割字符串
### syslog_pri
 * 简介：解析syslog中的pri（优先级）字段
### throttle
 * 简介：控制吞吐量
### tld
 * 安装：bin/logstash-plugin install logstash-filter-tld  
 * 简介：
### translate
 * 安装：bin/logstash-plugin install logstash-filter-translate  
 * 简介：
### truncate
 * 安装：bin/logstash-plugin install logstash-filter-truncate  
 * 简介：