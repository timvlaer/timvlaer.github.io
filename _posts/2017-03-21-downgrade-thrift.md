---
layout: post
title: "Downgrade Thrift on Mac using brew"
description: "Install thrift 0.9.3"
date: 2017-03-21
tags: [mac]
comments: false
share: true
---


With *Brew* you get the latest version of thrift installed. If you want to install a specific version, look for the correct formula in [the history of the thrift script](https://github.com/Homebrew/homebrew-core/commits/master/Formula/thrift.rb).

To install thrift 0.9.3, execute the following:

```bash
brew unlink thrift
brew install https://raw.githubusercontent.com/Homebrew/homebrew-core/9d524e4850651cfedd64bc0740f1379b533f607d/Formula/thrift.rb
```

