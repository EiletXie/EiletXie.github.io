---
title: Ajax，json多数据传输以及解析
date: 2019-08-02 13:51:30
tags:
- web
categories:
- JAVA
toc: true
---

# Ajax，json多数据传输以及解析

在一次表单传值中，我遇到了一个这样的表单提交情形，这个表单 有一些 字段 ，这些字段 name 对应了 一个 A对象的 属性名称

有两个内联的 table，有一个 不属于 A对象的 reason字段。我该如何传入呢？ 
<br/>

## 数据组成结构

**我们 要传输 一个表单对象，两个 table里面的 参数，一个普通字符串** 

于是 我就将其用过 JSON.stringify 一次性全部封装
<!--more-->

​          {% asset_img b1.jpg  数据组成 %}



我从表格中取得了 blameList 和 outBlameList；

```javascript
// 将责任工序表格数据传入后台
var blameTrList = $("#blameProcesses_table tbody").children("tr");
var blameList = new Array();
for (var i = 0; i < blameTrList.length; i++) {
var blameProcess = new Object;
var tdColumn = blameTrList.eq(i).children("td");
var operation_description = tdColumn.eq(0).text();
var standard_operation_code = tdColumn.eq(1).text();
var blame_content = tdColumn.eq(2).text();
var blame_type = tdColumn.eq(3).text();
blameProcess.blame_type = blame_type;
blameProcess.blame_content = blame_content;
blameProcess.operation_description = operation_description;
blameProcess.standard_operation_code = standard_operation_code;
blameList.push(blameProcess);
}
```

将所有数据传入

```javascript
 // 客诉对象数据
$.ajax({
    url: contextPath + "/complaint/editComplaint",
    type: "POST",
    dataType:"json",
    contentType:"application/json",
    cache:false,
    data:JSON.stringify( {
        "complaint": $("#complaintForm").serializeArray(), // 最初用的 是 serialize方法
        "reason": $("#reason_update_input").val(),
        "blameList": blameList,
        "outBlameList": outBlameList,
        "base_uid" :  $(this).attr("edit-id")
    }),
    success: function (result) {
        alert("更新成功！");
        window.location.reload();
    }
});
```
<br/>
##  Spring框架取值

我们需要一个 好用的插件 fastjson,它将 json通过数据类型 分为了 String ， JSONObject，JSONArray这三种类型。

也可以使用Java中的jackson，操作基本一致

```java
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.54</version>
</dependency>
```

在后台中 我们用一个 JSONObject来进行 前端参数的接受，注意 要用 @requestBody 进行标注

再取值的过程中 注意 先在前端打印出来 看看数据的情况，



我之前误以为 $("#complaintForm").serialize() 在后台拆分时肯定被封装成了一个对象了，所以我试图将其强转过来，

报了  complaint is not cast  不能强转的错误，在打印的过程中我才发现，serialize在 json封装后就直接成了 String类型，无法转变过来，

只能用 $("#complaintForm").serializeArray()  让json封装后成一个 JSONArray 这样才能取到字段值

```java
@ResponseBody
@PostMapping("/editComplaint")
public Msg editComplaint(@RequestBody JSONObject jsonObject, HttpSession session) throws Exception{
     User user = (User) session.getAttribute("user");
    String reason = jsonObject.getString("reason");
    JSONArray complaint = jsonObject.getJSONArray("complaint");

    System.out.println(complaint);
    List<BlameProcess> blameProcessList = jsonObject.getJSONArray("blameList").toJavaList(BlameProcess.class);
    List<BlameProcess> outBlameProcessList = jsonObject.getJSONArray("outBlameList").toJavaList(BlameProcess.class);
    // 新建一个现在的complaint
    CustomerComplaint nowComplaint = new CustomerComplaint();
    nowComplaint.setBase_uid(jsonObject.getString("base_uid"));
    nowComplaint.setModel(complaint.getJSONObject(0).get("value").toString());
    nowComplaint.setCustomcode(complaint.getJSONObject(1).get("value").toString());
    // 这里我们已经将所有的数据取出来了
    .........//省略业务逻辑
    return Msg.success();
}
```

这里我们算是会了 复杂的数据如何传值到后台解析的过程 。
<br/>

## 原理

那么分析一下 为什么 JSONObject 和 JSONArray 能 快速接受数据传入和 数据转换，我也是第一次遇到这种业务情景以及使用 fastjson ，对于它里面的原理有点困惑。

从 两个类的 字段属性 和  继承关系中 ，JSONObject是 用 Map作为底层数据结构，所以在取得对象时我们可以进行 getString(key),get(key) 获取 Object 的字段属性

{% asset_img b2.jpg  b2 %}

在 JSONObject 除了许多类型校对的 方法 和 原生的 get（key) ,move(key) ，有两个方法需要注意一下

{% asset_img b3.jpg  b3 %}

特殊一点的， getJSONObject 方法也是 通过 map.get(key) 获取的，只是它会进行 类型判断 instaceOf ,将数据类型强转成 JSONObject

getJSONArray 与  getJSONObject一致，只是 最后强转类型不同罢了。

而JSONArray不一样。JSONArray 以 List为 底层数据结构。

{% asset_img b4.jpg  b4 %}

通过List<Object>可以看出， list中存放的都当做Object对象，getJSONObject中取值 直接进行强转即可。 

{% asset_img b5.jpg  b5 %}

