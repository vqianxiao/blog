---
layout: post
title: Gradle快照包不能更新问题
category: 问题记录
description: Gradle快照包不能更新问题
tags: 问题记录
date: 2024/10/30 18:00:10
---
使用Gradle构建时，发现有个依赖更新了，但是本地的代码还是老的代码，刷新也没用，刷缓存重启也没有效果。之前一直使用maven没有发现这个问题，是因为gradle为了加快构建的速度,对jar包默认会缓存24小时，缓存之后就不在请求远程仓库了。

在Gradle里设置本地缓存的更新策略
```gradle
configurations.all {  
	// check for updates every build   
	resolutionStrategy.cacheChangingModulesFor  0,'seconds'  
}
```

但是我试了没有效果，发现这个插件效果没有应用到子项目里，参考地址：https://github.com/spring-gradle-plugins/dependency-management-plugin/issues/38
```gradle
dependencyManagement {
    resolutionStrategy {
        cacheChangingModulesFor 0, 'seconds'
    }
}
```
这样就可以了
