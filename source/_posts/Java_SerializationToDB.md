---
title: Java 序列化对象写入数据库
date: 2021/1/7 15:19:52
categories:
- 笔记
tags:
- java
- mysql
---

有时我们需要存储一个对象至数据库已达到持久化的目的，需要在数据库中设置特定数据类型的字段，并对实体设置属性存放流。

<!-- more -->

## 示例

### 实体类 WaifuData

```Java
package com.otaku.model;

import java.io.Serializable;
import java.util.Arrays;

public class WaifuData implements Serializable {
    private Integer wid;
    private byte[] priceFlow;

    //省略 getter/setter/constructor/toString
}
```

这里设置名为 priceFlow 的 byte[] ，对应数据表关系如下（mysql 5.7）：

| 字段名     | 类型     |
| ---------- | -------- |
| wid        | int      |
| price_flow | tinyblob |

### 实现

```java
//将流转化为对象
Queue<Float> priceFlow = null;
try{
    bis = new ByteArrayInputStream(data.getPriceFlow());
    ois = new ObjectInputStream(bis);
    priceFlow = (Queue<Float>) ois.readObject();
}catch (Exception e){
    throw new RuntimeException();
}
if (priceFlow.size() == 10){
    priceFlow.remove(); 
    priceFlow.add(waifuRepo.findByWid(data.getWid()).getPrice());
}else {
    priceFlow.add(waifuRepo.findByWid(data.getWid()).getPrice());
}

//将对象转化为流，写入
ByteArrayOutputStream bos = null;
ObjectOutputStream oos = null;
try{
    bos = new ByteArrayOutputStream();
    oos = new ObjectOutputStream(bos);

    oos.writeObject(priceFlow);
    oos.flush();
}catch (Exception e){
    throw new RuntimeException(e);
}finally {
    if(bos!=null){
        try{
            bos.close();
        }catch (Exception e) {
            System.out.println(e.getMessage());
        }
    }
    if(oos!=null){
        try{
            oos.close();
        }catch (Exception e) {
            System.out.println(e.getMessage());
        }
    }
}
//将流设置给data
data.setPriceFlow(bos.toByteArray());

//将流设置给data，并保存
data.setPriceFlow(bos.toByteArray());
waifuDataRepo.save(data);
```

