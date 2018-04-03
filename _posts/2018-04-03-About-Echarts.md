---
layout:     post
title:      echarts相关的一些总结
date:       2018-04-03
summary:    echarts
categories: jekyll pixyll
---

这里记录一下个人在日常工作中，使用echarts遇到的一些问题和处理方案。

### series.data.value为null、undefined、NaN时出现的渲染异常

这个问题如果开始知道数据就是这些异常数据的话，就会很快知道怎么解决，但比较麻烦的是在出错的时候，错误信息根本不会指向series.data有异常，例如我遇到这个问题的时候，因为在echarts中使用了渐变色，出来的错误信息反而提示渐变色设置有异常，然后就看到BoundRect的宽高全是NaN。最后没办法，只好打印了设置的option才发现data有异常。

解决方法：很简单，做一下data的异常检查就ok

### 先设置option为空对象，后来再设置具体的option对象，会出现柱状图无法渲染的异常

这里面可能存在以下排查困难：

1. 我自己认为后设置的option会按照正常的渲染效果对option进行合并；
2. 由于项目使用的是react，无论从打印出来的option，或者在开发者工具中查看图表的props，设置都是正确的，以为是其他代码写错了；

最后还是获取了DOM中的chart对象，通过chart.getOption()，仔细看了xAxis、yAxis、series的配置，发现xAxis.type、yAxis.type都设置为value，这才定位到问题。

起因是给option设置了一个空对象，造成echarts初始化xAxis.type和yAxis.type时，都设置成了‘value’，不知道这个算不算echarts的bug。然后自己在设置具体的option对象时，由于都没有显式设置上面两个值，导致在option合并的时候，将旧option的脏数据合并到了后设置的option，结果就悲剧了。

解决方法：

1. 设置setOption方法中的notMerge参数为true，这样子echarts就会用传入的新的option覆盖旧的option，这样子就不会有问题了；
2. 不要给option设置空对象。一般这种情况我们都是延迟设置series里面的data值，所以可以先把其他设置在option中设置好，包括xAxis、yAxis、tooltip属性等，data赋值空数组就好。然后等到需要的时候再更新data值，就不会有该问题了。

### 调试echarts一个有效的方法

容易想到，就是在页面渲染完之后，通过chart.getOption()方法，拿到echarts实际渲染使用的option，然后把他放到echarts官网示例中，看看哪里出了问题，大不了就是一项一项对照着文档检查常用配置，例如xAxis、yAxis、series里面的属性值