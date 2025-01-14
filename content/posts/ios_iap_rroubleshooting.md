+++
title = 'iOS 内购问题排查'
date = 2025-01-14T22:56:54+08:00
draft = false
+++

## 1. 获取不到商品信息

- ✅ 检查 AppStore Connect 有正确签署付费协议，或者确认付费协议在有效期内，可在 [AppStore Connect / 商务](https://appstoreconnect.apple.com/business) 中了解；
- ✅ 检查是否有正确填写内购商品的 **「产品 ID」** ，前往 **「AppStore Connect / 对应App下的分发 / 营利 - App内购买项目」** 中，确认所使用的 **产品 ID** 是已添加的，并且是以下状态之一：**准备提交**、**正在等待审核**、**审核中**、**已批准**；
- ✅ 检查是否使用真机测试，在模拟器中是无法使用内购功能；
- ✅ 检查测试机是否有使用正确的沙盒账号，前往 [AppStore Connect / 用户和访问 / 沙盒 / 测试账户](https://appstoreconnect.apple.com/access/users/sandbox) 校对；
- ✅ 检查 Bundle ID 是否正确
- ✅ 检查签名证书是否正确
- ✅ 检查商品创建是否有一小时及以上，内购商品创建后需要一定时间才能生效
- ✅ 检查设备时间是否已和世界时间同步

## 2. 购买失败

- ✅ 检查设备是否有限制内购
- ✅ 检查同一个商品是否存在未确认/未完成的订单
- ✅ 检查当前网络是否正常
- ✅ 检查时间是否已和世界时间同步

## 3. 服务端验证失败

- ✅ 检查是否将沙盒环境的订单票据用于正式环境，AppStore 的审核人员是使用沙盒环境测试内购；
- ✅ 检查调用接口的私钥是否正确
- ✅ 请求签名是否正确

## 额外

AppStore 内购回调数据结构：[App Store Server Notifications V2](https://developer.apple.com/documentation/appstoreservernotifications/app_store_server_notifications_v2)
