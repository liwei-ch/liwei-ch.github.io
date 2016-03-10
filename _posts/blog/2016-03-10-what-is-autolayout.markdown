---
layout: post
title: 什么是自动布局（auto layout）
description: 
category: blog
---

***什么是auto layout***

app界面由一系列视图（views）构成，视图体系中的views有相对的位置和大小等关系，即所谓的布局。
而自动布局技术(auto layout)可以允许设置约束，来保持views的布局，这种约束是一种线性关系。
app运行时views体系的内部和外部变换时，自动布局技术能动态根据这些线性关系计算出views的大小和位置，
从而保持views的布局。
