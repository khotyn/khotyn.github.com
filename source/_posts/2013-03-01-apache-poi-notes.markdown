---
layout: post
title: "使用 POI 解析 Excel 的几个注意点"
date: 2013-03-01 22:35
comments: true
categories: [Programming, Java]
---

最近在写一个解析 Excel 的程序，需要把从前端上传上来的 Excel 程序解析成 JSON 格式返回给前端，期间也试过 [jxl](http://jexcelapi.sourceforge.net/) ，不过它只支持到 Excel 2003。后来转而使用 apache 的 [POI](http://POI.apache.org/)，作为初次使用者，使用过程中遇到了不少的问题，在这篇博客中记录一下：

### 如何引入 XSSF

刚下手写的时候，直接使用了 HSSFWorkbook 处理 Excel，后来发现它只能处理 2003 的 Excel，而不能处理 2007 版本的 Excel，翻了一下 POI 的文档，发现处理 2007 的 Excel 需要使用 XSSFWorkbook，但是引入的 POI 包中却找不到 XSSF 相关的类，原猜想是因为引入了版本较新的 POI，而官方的文档还是比较老的，因而被误导。结果发现使用 XSSF 需要额外引入 **poi-ooxml** 这个 jar 包， XSSF 相关的类都在这个 jar 包中，mvn 依赖如下：

{% codeblock lang:xml %}
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>3.9</version>
</dependency>
{% endcodeblock %}

### 兼容处理 Excel 2003 和 2007

引入了 **poi-ooxml** 以后就使用 XSSF 来处理，本以为 XSSF 既然能够处理 Excel 2007，那么 2003 也该能处理吧？得向下兼容不是？没想到不行！总不能用扩展名来区别是 2003 还是 2007 吧，后来发现了 **WorkbookFactory** 这个类，它的 `create(InputStream inp)` 方法可以根据版本来选择创建 `HSSFWorkbook` 还是 `XSSFWorkbook`，使用者无需关心版本的问题，非常方便！

### 以 String 的形式读取单元格的数据

要读取的 Excel 中有一些数据是数字的，单元格的类型是 Numberic 的，而我希望将单元格中的数据都以 String 的形式拿出来。于是当我看到 `Cell.getStringCellValue()` 这个方法的时候满心欢喜，但是当用了之后，这货居然直接给我一个异常，定眼一样，这方式是 get**StringCell**Value，而不是 get**CellString**Value，真是瞎了眼了。事实上，Cell 也根本没有 `getCellStringValue` 这样的方法，后来借助了 [SOF](http://stackoverflow.com/questions/1072561/how-can-i-read-numeric-strings-in-excel-cells-as-string-not-numbers-with-apach)，查到了一个方法：

{% codeblock lang:java %}
row.getCell(j).setCellType(Cell.CELL_TYPE_STRING);
cellValue = row.getCell(j).getStringCellValue();
{% endcodeblock %}

是的，就是先把单元格变成 String 类型的，然后再调用 `getStringCellValue()` 来获取数据，虽然这方法可行，但是总感觉有点猥琐，不知大家有没有更好的方法？

### 错误地使用 `getPhysicalNumberOfCells()`

遇到的最后一个坑就是使用了 `getPhysicalNumberOfCells()` 这个方法来获取一行中单元格的数量以对行的中单元格进行遍历，依靠这个方法来遍历会出现的情况就是如果行中间的某些单元格是空的，那么你就解析不到这一行最后的几个单元格，原因是因为按照文档的说明

> Gets the number of defined cells (NOT number of cells in the actual row!). That is to say if only columns 0,4,5 have values then there would be 3.

这个方法只对有值的单元格进行计数，正确的遍历方法应该用 `getFirstCellNum()` 和 `getLastCellNum()` 来确定第一个单元格和最后一个单元格的位置，然后进行遍历，当然，要注意中间可能会出现空单元的情况，小心 NPE 异常。

另外，我发现很多 POI 的中文介绍资料上遍历 Excel 的样例代码都是采用了 `getPhysicalNumberOfCells()`，估计作者也没有经过彻底的测试，或者仔细阅读官方的文档和例子，平时大家做开发的时候，还是尽量找官方的介绍资料为好，看二手资料，一不小心就踩坑了。