---
title: paypal,google iap,stripe 循环订阅plan所能创建个数的最大值
date: 2018-11-05 11:02:25
tags: 
    - paypal
    - google iap
    - stripe
categories: 支付相关
---
## 前言
之前有个支付的修改，会涉及到建立非常多的plan 循环订阅plan。
所以就想知道 paypal stripe 和 google 这三个第三方支付对 plan 的创建有没有最大允许值。
## paypal
相关资料：[is-there-a-limit-to-how-many-billing-plans-can-i-create](https://stackoverflow.com/questions/41307481/is-there-a-limit-to-how-many-billing-plans-can-i-create)
看了一下应该是没有限制的:
```
I was told by Paypal support that there is no limit on how many billing plans you have created.
```
## google iap
相关资料： 
- [play-store-in-app-purchase-product-maximum-limit](https://stackoverflow.com/questions/16098287/play-store-in-app-purchase-product-maximum-limit)
- [in-app-subscription-is-there-a-limit-of-inventory-queries-from-apps](https://stackoverflow.com/questions/24091810/in-app-subscription-is-there-a-limit-of-inventory-queries-from-apps)

看了一下，应该也是没有限制的
```
There is no limit in Google Play, or if there is it is not publicly documented.
On another note, why would you possible need 10000 in app products?
```
## stripe
看了一下文档，也没有找到，应该也是没有限制的。





