---
title: The Bulding of Misaka Gate
date: 2016-12-01
---

# Backgrounds

[二次元之门](http://2d-gate.org) 是台湾一个可以线上看动漫的论坛。 然而这个论坛并没有对移动设备做过优化。这就导致了在手机上追剧会出现：在 5 寸的屏幕上，用笨拙的手指点按不到 1 平方厘米的选集 tab 的尴尬场面。于是萌生了~~搞个大新闻~~造个 app 的想法。

# Toolchain

* 本地编写 --> Push changes to GitHub --> Travis-CI 持续集成 --> 如果构建通过并且此次commit有tag，则上传 signed apk 到 GitHub release 中。
* 使用~~友盟~~ Firebase 来跟踪bug。
友盟优势：可以送达终端上无 Google Play Services 的用户。
友盟缺点：需要加敏感权限（电话）。
Firebase优势：不用权限，谷歌是人类的希望。
Firebase缺点：build.gradle 里需要加一行 plugin，墙内用户及无 Google Play Services 的用户无法送达。
* 使用 Rx 来进行异步操作
  使用 DBFlow 来简化数据库操作

# Continuous Integration

### 签名
发 release 时需要用 keystore 签名，但显然既不能把 keystore 添加到版本管理中，也不能把 keystore 密码加到任何公众可见的地方。幸运的是，我们可以用 [Travis file encryption](https://docs.travis-ci.com/user/encrypting-files/) 来加密 keystore，并将私钥以及 keystore 密码存到 CI 的环境变量中，在签名时用私钥解密文件来签名。

```Groovy
    signingConfigs {
        release {
            File signingPropFile = rootProject.file('signing.properties')
            if (signingPropFile.exists()) {
                Properties signingProp = new Properties()
                signingProp.load(signingPropFile.newDataInputStream())
                storeFile file(signingProp.get("release.storeFile"))
                storePassword signingProp.get("release.storePassord")
                keyAlias signingProp.get("release.keyAlias")
                keyPassword signingProp.get("release.keyPassword")
            } else if (System.getenv('encrypted_046cd0986819_key') != null && System.getenv('encrypted_046cd0986819_iv') != null) {
                storeFile rootProject.file('gate.jks')
                storePassword System.getenv("keyStorePassword")
                keyAlias System.getenv("keyAlias")
                keyPassword System.getenv("keyPassword")
            }
        }
    }
```

### Travis 自动发布

```yml
deploy:
  provider: releases
  api_key:
    secure: yavm4GgZaKa3mzFFRIwysDzFLrcLQo1epelzo0HnSBTpqKRtH/RtY6Ze8Scpsp15kM2Sj8ut0UEC7IZPxnG6oMYqwofRLvmHslxwJYIIf5iekOnblNhitt1Y5IgikCD3KIFmlC00276ktowYLXuhe3skSuV6WL6R/SJKV758FSfWwc/N2CSAKZn0GNvIzWNVcd5WYBJg2gHO3qB04hF93F/B7wbbfQWUoni7FvzXjtZV5wki68O45BS31hExU+AiVGtVNczcmTDgReo/c6TWESTFQsf37TtgG0uUY6XnGvCW2yzH0MbbgTEHhVMCNY97pb9pF/QwwyE3gjvL2e1Q57h1Vf/gxPLXVQ3H9pDeTilDNQ4ETYMyP0MefVgu2dxZG9U2QzLF2gJvfyfxSYs/drLVflyEJf8cu9OLcVslCjjeHWg5TzeX35UjCuBNAfFEO+cmETSoBnWg/C0AZI7Cls5p6Wb33Bu9czs0WwKej1DnWXE+DgCH1XJTEwix5ClumLAK5S2ZIVrTi7AeEs9Mi984EFXzqRv9VQnaBXD4NcY2rnD39ixZ2FP0r7pT24IHkTyye7EQC3ez6X2IbG1nfrPbGgPL58sWDiwRpgdZBHTTegSWpYsR4QAsVO1Lo48nEzK5iHKR6F0lp9f8drdGHWdIQ7GJvTOVzmwQpSZhC/0=
  file: "./app/build/outputs/apk/app-release.apk"
  skip_cleanup: true
  on:
    repo: jilulu/MisakaGate
    tags: true
```

值得注意得是 ```skip_cleanup``` 要设为 ```true```，不然刚刚 build 出来的 apk 可能已经被清理掉了。最后一行的 ```tags: true``` 确保日常 commit 不会发版，只有加过 tag 的会被发布到 GitHub 的 release 中。
<!-- more -->
