---
layout: post
title: "Simple introduction to database index"
description: ""
category:
tags: [mysql, index, optimize]
---


## Why using index

A good metaphor of index. Assume you have a 2000 pages book and you have to lookup a word in it. You need search all pages one by one in this book without index. With index, you don't. Index will tell you the word you looking for exists in page 299-301. Then you just need search these three pages.

## When to use it

Above metaphor is a good example of when you should add index to table. If you have a large set of data, index can help increase the query speed.

## When don't use it

Index is data too. It needs place to store and maintain. Let's say now you only have a book includes two pages. If you add index to this book. It needs one additional page to maintain and store index. And it can only indicates that the word you're looking for exists in page 1. Obviously. it doesn't help much but cost a lot.

Another circumstance is: 2000 pages book and index shows that the word exists in page 1-399, 400-799, ... 1799-2000. This index is totally useless. Actually. this should not be included in this section. This involved with your table structure. I can't say much about it. See [mysql docs](http://dev.mysql.com/doc/refman/5.5/en/optimization-indexes.html) for more detail please.

## Exam your index!

After add index to your table. Don't forget exam it actually used by mysql. [EXPLAIN](http://dev.mysql.com/doc/refman/5.5/en/explain-output.html) is very helpful here. I once added a multiple-columns index but it never get used because I missed a important column that involved with that query.
