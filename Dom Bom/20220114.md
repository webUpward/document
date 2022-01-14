[TOC]

# 笔记

## 节点类型
> 元素节点，文本节点，属性节点

## 节点操作
> 获取元素节点（元素ID document.getElementById ，标签名字 document.getElementsByTagName，类名 getElementsByClassName，class选择器 document.querySelector）

## 获取元素
> childNodes，该属性可用来获取一个元素的所有子元素，得到一个包含所有子元素的数组
example: document.getElementsByTagName(“body”)[0].childNodes
> parentNode, 获取当前节点的父节点元素，如果指定节点没有父元素则返回null
> nodeType, 元素节点的nodeType属性值是1, 属性节点的nodeType属性值是2, 文本节点的nodeType属性值是3

## 创建元素
> createElement 非DOM节点树上的组成部分，无法显示在浏览器里
> createTextNode 创建一个文本节点用于放文本内容
> appendChild 新创建的节点插入节点树