---
layout: post
title: "正则表达式的反向引用"
description: ""
category:
tags: ["regulr expression"]
---



<pre>
  s = "this is a test string"
  reg = /(is)/
  s.gsub reg, "anything"

  1.9.3-head :055 > $1
  => "is"

  1.9.3-head :055 > s.gsub reg, "\1 not"
   => "this is not a test string"
</pre>
