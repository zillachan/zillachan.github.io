---
layout: post
title: Gradle configuration & best-practice
date: 2017-11-10
categories: blog
tags: [Gradle,Android]
description: The gradle configuration & best practice.
---

## What's Gradle
Generally speaking, it's a build tool.
```
xxx
```
## Why we use Gradle
## How to use Gradle
- Custom your respositories(for chinese user)

```
allprojects {
    repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
        maven { url "https://jitpack.io" }
        maven { url 'https://dl.google.com/dl/android/maven2/' }
        google()
        jcenter()
    }
}
```
- Config library version

- xxx
## Advanced features














