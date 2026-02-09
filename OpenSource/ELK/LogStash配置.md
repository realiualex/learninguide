# LogStash 配置

## 插件安装

```
./logstash-plugin list
./logstash-plugin install logstash-input-azureblob
```

## 有关Logstash的配置解读

* Logstash 数据处理的三大步骤: Input --> Filter --> Output

* 如果logstash要处理多个业务的数据，可以用 logstash pipeline，每一个业务放到单独的pipeline文件夹下，一个pipeline文件夹一般只有一个input，output一般也只1个（当然如果业务想写到多个地方，也可以配置多个）。同一个业务里，如果有不同的服务，可以用不同的文件来处理 filter里的逻辑，然后将 add_field 加上一个字段表示这个服务

  

## filter Grok的示例

下面 filter Grok的示例中，其中第一部分 ?<api_path>\/api\/v1\/\S+) 表示的是使用正则进行处理，正则判断的是 /api/v1/xxx的这种路径，并将搜索到的结果保存到 api_path 中，这个结果可以再 output 中，通过 "%{[api_path]}" 的方式进行引用。

%{WORD} 是使用 grok 里的WORD pattern进行查找； BASE10NUM则是搜索数字，后面的:int指的是将数据类型转换为 int，默认是string类型（常用的还有 float, boolean, epoch，一共仅此4种）；NUMBER是在BASE10NUM 搜索的基础上，再加上对科学计数法的支持；最后的 (?\<unit\>(s|ms|us)) 指的是将 s 或者 ms，或者us 保存到 unit里

再if 语句后面有一个 else {} 表示，如果grok匹配不到，则删除

最后if _grokparsefailure 指的是，对于 grok匹配不到的数据，一般会加上这个tag，然后依然保留这个数据，我们通过将这些数据删除，能确保后续处理的数据，一定是grok 匹配到的

```

filter {
  if "/my-log-group" in [logGroup] {
    grok {
      match => {
          "message" => "(?<api_path>\/api\/v1\/\S+) %{WORD} %{BASE10NUM:code:int} %{NUMBER:duration}(?<unit>(s|ms|us))"
      }
    }
  }
else {
  drop {}
  }
if "_grokparsefailure" in [tags] {
  drop {}
  }
}

```

不过在上述的示例种，也不免发现，由于时间单位，既有秒，又有毫秒，还有微秒，我们在存储的时候，希望时间统一。此时最好的办法是让开发写日志的时候统一起来。如果开发没有处理，我们也可以通过嵌入ruby脚本来完成。比如:

```
filter {
  if "/my-log-group" in [logGroup] {
    grok {
      match => {
        "message" => "(?<api_path>\/api\/v1\/\S+) %{WORD} %{BASE10NUM:code:int} %{NUMBER:duration}(?<unit>(s|ms|us))"
      }
    }
    ruby {
      init => "
        def to_milliseconds(value, unit)
          case unit
          when 'ms'
            value
          when 's'
            value * 1_000
          when 'us'
            value / 1_000.0
          else
            value
          end
        end
      "
      code => "event.set('duration', to_milliseconds(event.get('duration').to_f, event.get('unit')))"
    }
  } else {
    drop {}
  }
  
if "_grokparsefailure" in [tags] {
  drop{}
  }
}
```




## 解读Filter常用部分，也是对logstash配置的关键部分
* 在 filter里，一般可以在 mutate 里对字段进行处理，最常用的是 split, add_field 和 join。其中 split 就是将一个 string切割成list，可以指定按某一个字符切割。add_field 就是对刚刚切割成的list里的某一部分进行提取，创建一个新的字段来保留。join 一般会将切割后的list，还原回string。
* 需要注意的是：**在一个 mutate block 里，并不是严格按照上下顺序执行的**，所以一般我们会将 split 和 add_field 翻到一个mutate里，将 join放到另外一个mutate里。如果一个mutate里有 split, add_field, join，很有可能会出现 add_field发生在 join之后

