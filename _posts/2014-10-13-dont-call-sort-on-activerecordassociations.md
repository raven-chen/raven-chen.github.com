---
layout: post
title: "Don't call sort! on ActiveRecord::Associations"
description: ""
category:
tags: [rails]
---


The reason is `sort!` acts same as `sort` on `ActiveRecord::Associations`.

<pre>
  # Source code from ActiveRecord::Associations
  # The <tt>@target</tt> object is not \loaded until needed. For example,
  #
  #   blog.posts.count
  #
  # is computed directly through SQL and does not trigger by itself the
  # instantiation of the actual post records.
</pre>

I called `sort!` on a passed in parameter which is `ActiveRecord::Associations` and expect it acts like `sort!` on Array. This makes a bug of that method. I think rails is better to remove `sort!` from this class's method list. `NoMethodError` is much better than this.
