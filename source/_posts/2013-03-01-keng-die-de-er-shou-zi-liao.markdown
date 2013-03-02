---
layout: post
title: "POI 解析 Exel 的几个注意点"
date: 2013-03-01 22:35
comments: true
categories: [Programming, Java]
published: false
---

最近在写一个解析 Excel 的程序，需要把从前端上传上来的 Excel 程序解析成需要的 JSON 格式的数据返回给前端，中间也试过 jxl 等类库，不过对 Excel 版本的支持不是很好。后来转向了使用 apache 的 poi，作为初次的使用者，其中有几次错误的使用了几个 API，在这篇博客中记录一下：

### 如何引入 XSSF

第一次写 poi 的时候，直接使用了 HSSF 处理 Excel，后来发现 HSSF 只能处理 2007 的 Excel，而不能处理 2007 版本的 Excel，翻了一下 poi 的文档，发现处理 2007 的 Excel 需要使用 XSSF 和不是 HSSF，但是发现引入的 poi 包中却找不到 XSSF 相关的类，原以为是因为引入了版本较新的 poi，而官方的文档还是比较劳的，后来发现使用 XSSF 需要额外引入 **poi-ooxml** 这个 jar 包， XSSF 相关的类都在这个 jar 包中，mvn 依赖如下：

{% codeblock lang:xml %}
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>3.9</version>
</dependency>
{% endcodeblock %}

### 兼容处理 Excel 2003 和 2007

引入了 **poi-ooxml** 以后就使用 XSSF 来处理，本以为 XSSF 既然能够处理 Excel 2007，那么 2003 也该能处理吧？没想到不行！总不能用扩展名来区别是 2003 还是 2007 吧，后来发现了 **WorkbookFactory** 这个类，它的 `create(InputStream inp)` 方法根据版本来选择创建 `HSSFWorkbook` 还是 `XSSFWorkbook`，使用者无需关心版本的问题，非常方便！

### 以 String 的形式读取单元格的数据

因为要读取的 Excel 中有一些数据是数字的，但是程序解析 Excel 希望将单元格中的数据都以 String 的形式拿出来，于是，当我看到 `Cell.getStringCellValue()` 这个方法的时候满心欢喜，但是当我用了之后这货居然直接给我一个异常，定眼一样，这方式是 get**StringCell**Value，而不是 get**CellString**Value，真是瞎了眼了。事实上，Cell 也根本没有 `getCellStringValue` 这样的方法，后来借助了 SOF，查到了一个方法：

{% codeblock lang:java %}
row.getCell(j).setCellType(Cell.CELL_TYPE_STRING);
cellValue = row.getCell(j).getStringCellValue();
{% endcodeblock %}

是的，就是先把 Cell 变成 String 类型的，然后再调用 `getStringCellValue()` 来获取数据，虽然这方法可行，但是总感觉有点猥琐，不知大家有没有更好的方法？

### 错误地使用 `getPhysicalNumberOfCells()`

遇到的最后一个坑就是遍历 Row 的时候用了 `getPhysicalNumberOfCells()` 这个方法来获取一行中单元格的数量以进行遍历，依靠这个方法来遍历的会出现的情况就是如果你一列中中间的某些单元格是空的，那么你就永远解析不到最后的几个单元格，原因是因为按照文档的说明

> Gets the number of defined cells (NOT number of cells in the actual row!). That is to say if only columns 0,4,5 have values then there would be 3.

这个方法只对有值的单元格进行计数，正确的遍历方法应该用 `getFirstCellNum()` 和 `getLastCellNum()` 来确定第一个单元格和最后一个单元格的位置，然后进行遍历，当然，要注意中间可能会出现空单元的情况，小心会出现 NPE 异常。

另外，我发现很多 poi 的中文介绍资料上遍历 Excel 都是采用了 `getPhysicalNumberOfCells()`，估计作者也没有经过什么彻底的测试，或者仔细阅读官方的文档和例子，平时大家做开发的时候，还是尽量找官方的介绍资料为好，看二手资料，一不小心就踩坑了。