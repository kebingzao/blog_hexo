---
title: web 上设置图片不能拖拽
date: 2018-05-03 14:51:35
tags: 前端相关
---
### web上的图片如果要设置成不能拖拽，有两种情况


#### 一种是使用css， 让其不能选择
{% codeblock lang:css %}
.i-user-select-none{
  -webkit-user-select:none;
  -moz-user-select:none;
  -khtml-user-select:none;
  -o-user-select:none;
  user-select:none;
}
{% endcodeblock %}



#### 一种是js，将 onselectstart ， ondragstart 方法return false
{% codeblock lang:js %}
$(img).attr({
    "onselectstart": "return false;",
    "ondragstart": "return false;"
});
{% endcodeblock %}