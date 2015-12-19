---
layout: post
title: "Rubyer learning Go"
description: ""
category:
tags: []
---


It's time to study a compile language! Go

## No class but package

## Handle error everywhere

## Go package Reflect

## Questions

> *string means ?
allow this field to be nil

> import . "github.com/go-sql-driver/mysql" // what does "_" means ?
means only invoke init() of the package but not include it

> Array... means ?
open and assign array elements. like in ruby:  a,b,c = [1,2,3]

> value.(Injector)
Injector is an interface, this expression means if value implemented Injector's functions

    type Context struct {
      *qor.Context
      *Searcher
    }

Mixin Searcher into Context. You can call Context.search(search is defined in Searcher) directly

## Pointer study

`func *(string) memoryLocation` and `func &(string) pointer`

## Interface

    type Shape interface {
      area()
    }

    type Circle struct {
    }

    func (c *Circle) area() {
      fmt.Println("I'm a circle")
    }

    type Rectangle struct {
    }

    func (c *Rectangle) area() {
      fmt.Println("I'm a rectangle")
    }

  In this example, Circle and Rectangle can be used as "Shape" together. like this

    func totalArea(shapes ...Shape) {
      for _, s := range shapes {
        s.area()
      }
    }

    totalArea(&c, &r)

Kind of `duck typing` in Go...



