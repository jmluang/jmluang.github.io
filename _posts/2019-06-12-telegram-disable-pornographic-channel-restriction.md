---
layout: post
title: telegram-disable-pornographic-channel-restriction
---

> “自己动手，丰衣足食” -- 毛泽东同志

因为 App Store 的内容审核政策，Telegram iOS 很多群组都被 block 了。虽然可以通过安装 5.0.4的ipa 绕过，但是版本太久了，许多新的功能都不能用，十分蛋疼。不过没关系，因为 Telegram 的代码开源在了 github 上，所以这点问题也不是什么大问题。

# 下载代码

先把代码下载到本地，项目地址是 https://github.com/peter-iakovlev/Telegram-iOS。

` git clone --recurse-submodules git@github.com:peter-iakovlev/Telegram-iOS.git `

下载的同时可以安装一些依赖库 `yasm` 和 `pkg-config` ，用 `brew` 安装即可

` brew install yasm pkg-config `

# 申请 Api Key

在编译之前，你需要在 https://my.telegram.org/auth 申请一个 API Key 以和服务器进行通讯。

申请后，打开文件 `Telegram-iOS > Telegram-iOS > Config-Fork.xcconfig`
替换你申请的 `APP_CONFIG_API_ID` 和 `APP_CONFIG_API_HASH`。

![Config-Fork.xcconfig]({{site.url}}/images/posts/2019/2019061201.png)

```
GCC_PREPROCESSOR_DEFINITIONS = $(inherited) APP_CONFIG_API_ID=*your_id* APP_CONFIG_API_HASH="\"*your_hash*\"" APP_CONFIG_HOCKEYAPP_ID="\"\""
```

直接替换即可

# 运行代码

首先选择 target 为 Telegram-iOS-Fork。

![选择指定的 Scheme]({{site.url}}/images/posts/2019/2019061202.png)

点击编译，等待一段时间后，就可以看到 Build Successed 了！

接下来我们开始修改代码

# Disable restriction

打开文件 `TelegramCore > TelegramCore > Objects > Peers > PeerUtils.swift`，找到函数 `restrictionText`，这个就是判断 restriction 的函数，我们直接返回 `nil` 就可以了

![PeerUtils.swift]({{site.url}}/images/posts/2019/2019061203.png)

```swift
# before
public var restrictionText: String? {
    switch self {
        case let user as TelegramUser:
            return user.restrictionInfo?.reason
        case let channel as TelegramChannel:
            return channel.restrictionInfo?.reason
        default:
            return nil
    }
}
# after
public var restrictionText: String? {
    return nil
    switch self {
        case let user as TelegramUser:
            return user.restrictionInfo?.reason
        case let channel as TelegramChannel:
            return channel.restrictionInfo?.reason
        default:
            return nil
    }
}
```

再次Build，进入一个被 block 的频道，Let's Fly！

# 编译到 iPhone 上

## 关闭 Siri 授权
如果是免费账号的话，需要通过修改代码关闭 Siri 的授权申请，否则运行时会白屏卡住。

打开文件 `Telegram-iOS > Telegram-iOS > AppDelegate.swift`，找到获取 Siri 权限的代码 `siriAuthorization`，直接返回拒绝状态即可。

![AppDelegate.swift]({{site.url}}/images/posts/2019/2019061204.png)

```swift
# before
siriAuthorization: {
    if #available(iOS 10, *) {
        switch INPreferences.siriAuthorizationStatus() {
            case .authorized:
                return .allowed
            case .denied, .restricted:
                return .denied
            case .notDetermined:
                return .notDetermined
        }
    } else {
        return .denied
    }
}

#after
siriAuthorization: {
    return .denied
    if #available(iOS 10, *) {
        switch INPreferences.siriAuthorizationStatus() {
            case .authorized:
                return .allowed
            case .denied, .restricted:
                return .denied
            case .notDetermined:
                return .notDetermined
        }
    } else {
        return .denied
    }
}
```

## 使用自己的证书签名

选择 Telegram-iOS，对于所有的 targets，在 signing 区块中勾选 Automatically manage signing。

![signing]({{site.url}}/images/posts/2019/2019061205.png)

然后在每一个 targets 的 Build settings 中的 signing 区块中清空 Code Signing Entitlements。

## 修改 Bundle Identifier

把每一个 target 的 Bundle Identifier 都改掉。除了 Telegram-iOS 之外，其他的 target 的 Bundle Identifier 应该以 Telegram-iOS 的 Bundle Identifier 为前缀。比如如果 Telegram-iOS 的新 Bundle Identifier 是“free.telegram”，则 Share 的新 Bundle Identifier 应该设置为“free.telegram.Share”。

![Bundle Identifier]({{site.url}}/images/posts/2019/2019061206.png)

除此之外，还需要去每一个 target 里的 build settings 选项卡中，修改 User-defined 下的 APP_BUNDLE_ID 为 Telegram-iOS 的 Bundle Identifier：

![APP_BUNDLE_ID]({{site.url}}/images/posts/2019/2019061207.png)

## 修改 App Group

在 Telegram-iOS 的 `capabilities -> App Groups` 中新建一个 App Groups，名字必须与上面设置的的 Bundle identifier 保持一致（比如我的是 free.telegram ），否则运行时会黑屏出现 Error 2 错误。然后关闭 App Groups 功能再重新打开，然后选择你刚刚新建的项目。接下来的每一个 target 都需要先关闭 App Groups 功能再重新打开，然后选择你刚刚新建的项目。

![App Group]({{site.url}}/images/posts/2019/2019061208.png)

除此之外，你还需要前往每一个 target 的 build settings，然后将 User-defined -> Provisioning profile 清空。否则 App Groups 列表下面会有警告。

![Provisioning profile]({{site.url}}/images/posts/2019/2019061209.png)

# 运行

在完成了以上步骤之后，我们就可以运行了。经过编译的过程后，Telegram Fork 就安装到了你的手机上。为了运行你签名的应用程序，你需要在 iPhone 上前往设置->通用->描述文件，然后信任你自己的开发者证书。如果遇到问题，可以通过搜索解决

# 常见问题

## 打开后白屏
有可能是位置的问题，可以尝试在 `Capabilities -> Background Modes -> 勾选 Location updates ` 解决。

## 黑屏 Error 2
查看上文的 修改 App Group 解决

# 总结

这种方法的优点就是可以享受最新的功能，不过缺点也是很明显的，需要定期“续费”，个人的证书大概7天左右就会过期，这个时候需要重新 build 一次。
