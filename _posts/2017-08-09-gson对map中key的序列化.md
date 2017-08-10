---
layout: post
title:  "gson对map中key的序列化与发序列"
date:   2017-08-09
categories: gson
---

## 0x01 问题
```java
private final Map<AutoKey, AutoKey> map = new HashMap<>();
...
Gson gson = new Gson();
String json = gson.toJson(map.values());
```
这段代码反序列化之后json串的格式是这样的。

		{"(room,0,4096,0)": {"name": "room", "step": 4096, "current": 0, "initValue": 0}}

key和value是相同的对象，但是序列化后的json串却看到少了对象中每个字段的name。


## 0x02 解决办法
两种思路 <br>
1、存set，这样话序列化就是正常的了。
2、查明到发生这个问题的原因，开启enableComplexMapKeySerialization，并且定义好相应key类型的适配器就OK了。
```java
// it's ok
GsonBuilder gsonBuilder = new GsonBuilder();  
gsonBuilder.enableComplexMapKeySerialization();  
Gson gson = gsonBuilder.create();  

class AutoKeyAdapter extends TypeAdapter<Point> {  
  
    @Override  
    public void write(JsonWriter out, AutoKey value) throws IOException {  
        if (value == null) {  
            out.nullValue();  
        } else {  
            out.value("(" + value.getName() + "," + value.getStep() + "," + value.getCurrent() + "," + value.getInitvalue() + ")");  
        }  
    }
  
    @Override  
    public AutoKey read(JsonReader in) throws IOException {  
        if (in.peek() == JsonToken.NULL) {  
            return null;  
        } else {  
            String str = in.nextString();   
            str = str.substring(1, str.length() - 1); //根据writer的格式，解析字符串  
            String[] pair = str.split(",");  
            AutoKey ak =  new AutoKey();  
            ak.setName(Integer.parseInt(pair[0]));  
            ak.setStep(Integer.parseInt(pair[1]));  
			ak.setCurrent(Integer.parseInt(pair[2]));  
			ak.setInitvalue(Integer.parseInt(pair[3]));  
            return ak;  
        }  
    }  
} 

```

## 0x03 原因
不开启enableComplexMapKeySerialization选项，默认gson会将key的对象序列化为一个数组，所以在反序列化的时候无法将该数组反序列化为对象，会抛出 Expected BEGIN_OBJECT but was BEGIN_ARRAY 这个异常。
