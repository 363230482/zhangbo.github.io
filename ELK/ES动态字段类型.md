官方文档： [https://www.elastic.co/guide/en/elasticsearch/reference/7.x/indices-templates.html]  
可在设置日期格式等时，不用每个index手动设置
```
PUT host:port/_template/template_name
{
    "index_patterns": ["trace_collect*"],
    "order": 5, // 权重越大，越优先匹配默认为0
    "settings": {
        "number_of_shards": 1  // 副本数量
    },
    "aliases": {
        "alias_1": {} // 别名
    },
    "mappings": { // 自定义mapping
        "properties": {
            "timestamp": {
                "type": "date",           
                "format": "yyyy-MM-dd HH:mm:ss"
            },
            "uid": {
                "type": "keyword"
            }
        }
    }
}
```