---
title: excel转csv
date: 2017-05-24 20:40:11
tags: 工作总结
---
公司项目是在Excel文件中写入数据，然后将其转换成CSV文件。程序读取CSV文件中数据存放在内存中的。最近在处理这两种文件类型中有了一些经验，基于此想记录总结下。

### 机器环境 ###

首先说明本机环境:
	
```
windows 7 SP1版本, office 2016, jdk 1.8, python 2.7.13, 
javacsv 2.0, fastjson 1.1.46
```

### 编辑Excel文件的注意点 ###

1.Excel文件首行首列值不能为"ID"，否则打开转换生成的CSV文件会报错：

文件格式和拓展名不匹配。文件可能已损坏或不安全。除非您信任其来源，否则请勿打开。是否仍要打开它？

![](http://i.imgur.com/R1qFIbR.png)

2.Excel文件中不可随意进入单元格，如下图所示，否则转换生成的CSV文件中对应位置将以空字符呈现

![](http://i.imgur.com/CKZiNbL.png)

### Excel文件类型转换成CSV文件 ###

可以使用VBS脚步实现，也可以调用Python库实现。

1.VBS脚本

```
if WScript.Arguments.Count < 2 Then
	WScript.Echo "Please specify the source and the destination files.
	 Usage: ExcelToCsv <xls/xlsx source file> <csv destination file>"
		    Wscript.Quit
	End If
		
csv_format = 6
		
Set objFSO = CreateObject("Scripting.FileSystemObject")
		
src_file = objFSO.GetAbsolutePathName(Wscript.Arguments.Item(0))
dest_file = objFSO.GetAbsolutePathName(WScript.Arguments.Item(1))
		
Dim oExcel
Set oExcel = CreateObject("Excel.Application")
		
Dim oBook
Set oBook = oExcel.Workbooks.Open(src_file)
	
oBook.SaveAs dest_file, csv_format
		
oBook.Close False
oExcel.Quit
```

使用时可以将该脚本复制存为XlsToCSV.vbs文件，再直接进入到当前文件目录下的命令行里输入（ ">" 表示命令行操作环境）：

```
> XlsToCSV.vbs test.xlsx test.csv
```

上述方式仅仅能一次操作一个文件，可以使用如下批处理脚本读取文件下所有.xlsx文件，依次将其转移成.csv文件：

```
@echo off
for %%i in (excel\*.xlsx) do XlsToCSV.vbs %%i csv\%%~ni.csv
pause
```

上述bat脚本将循环读取当前文件夹下的excel文件下的.xlsx的类型文件将其转换成csv文件下的.csv文件。

2.python脚本
		
可以使用python的csvkit库快速进行Excel转CSV文件

首先需要搭建python环境，并且将下述路径添加到环境变量中（本机将python安装至默认路径C盘下）：
		
```
C:\Python27;C:\Python27\Scripts
```

然后在命令行进行csvkit库的安装:	

```
> pip install csvkit
```

安装需要一段时间。安装完成后，进入Excel文件所在文件夹，输入命令进行文件类型转换

```
> in2csv test.xlsx > test.csv
```

经测试，in2csv命令在转换时会默认使用utf-8编码，由于Excel打开默认以gbk编码打开，会导致中文乱码的情况。详细可查看其官方文档：

[https://csvkit.readthedocs.io/en/latest/](https://csvkit.readthedocs.io/en/latest/)

### java加载CSV文件数据 ###

使用javacsv jar包进行数据读取，可以直接前往官网下载，也可使用maven进行管理：

官网： [https://www.csvreader.com/java_csv.php](https://www.csvreader.com/java_csv.php)

maven依赖：

```
<dependency>
	<groupId>net.sourceforge.javacsv</groupId>
	<artifactId>javacsv</artifactId>
	<version>2.0</version>
</dependency>
```

使用主要方法如下：

```
String str[] = null;
CsvReader csvReader = new CsvReader(new FileInputStream(file), 
	Charset.forName("GBK"));//创建CsvReader对象，指定读取文件，并指明编码格式
csvReader.readHeaders();//跳过表头
while (csvReader.readRecord()) {//判断下一行是否有数据
	str = csvReader.getValues();//读取一行的每一格以字符数组方式返回
    csvReader.getRawRecord();//读取一行数据,返回字符串
    csvReader.get("Id");//读取某一列数据，返回字符串
} 
```

此处表头实则表示CSV文件内容第一行，如下图所示，表中第一行内容即为表头。

![](http://i.imgur.com/knQwIJK.png)

项目中使用到如此场景：上图第一行是内容描述，第二行是一个Java Bean 类的属性，第三行是类属性的类型，其后则是类的值。需要读取CSV文件数据并赋值于生成的java Bean类。

读取到的每一行数据将以key-value 形式保持在Json中，key代表类的属性，value代表类的属性值。再将json置于json数组中以保存。

注意本地Excel和CSV文件的默认编码格式是GBK。


```
创建两个成员变量字符串数组用于存储java bean类的属性与属性类型：
private String[] attributes;
private String[] attributeTypes;

读取每一行数据进行数据的分配：
String str[] = null;
CsvReader csvReader = new CsvReader(new FileInputStream(file), 
	Charset.forName("GBK"));//创建CsvReader对象，指定读取文件，并指明编码格式
csvReader.readHeaders();//跳过表头
int pos = 1;
JSONArray jsonArray = new JSONArray();
JSONObject jsonObject = new JSONObject();
while (csvReader.readRecord()) {	//判断下一行是否有数据
	if (pos == 1) {					//读取到第一行赋值给类属性
		attributes = csvReader.getValues();
	}else if (pos == 2) {			//读取到第二行赋值给类属性类型
		attributeTypes = csvReader.getValues();
	}else {
		str = csvReader.getValues();
		for (int i = 0; i < str.length; i++) {	//遍历数组，依次赋值类属性
			jsonObject.put(attributes[i], str[i]);
		}
		jsonArray.add(jsonObject);				//将json存储于jsonArray数组中
		jsonObject.clear();
	}
	pos++;
}
```

此处使用的fastjson jar包maven依赖：

```
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>fastjson</artifactId>
	<version>1.1.46</version>
</dependency>
```

再从JSONArray中解析出数据来。不过在解析之前，需要先获取该Java Bean类，基于类的属性创建类可以先创建该.java文件，然后向其写入类属性信息，大体如下方式：

```
writer.write("public class " + %ClassName% + "{\r\n");//ClassName代表类名，与.java文件名需要保持一致
for (int i = 0; i < attributes.length; i++) {//循环遍历属性值数组，依次写入文件流中
	if (attributesType[i].equals("string")) {
		attributesType[i] = "String";
	}
	//此处为方便，直接写入public属性
	writer.write("\tpublic " + attributesType[i] + " " + attributes[i].substring(0, 1)
		.toLowerCase() + attributes[i].substring(1) + ";\r\n");
}
writer.write("}");
```

最后从JSONArray中解析获得ClassName类的对象

```
%ClassName% name[] = null;//创建该类类型数组用于存放从csv文件中读取的数据，
	对应关系是数组的一个值即一个%ClassName%引用与CSV文件中的一行数据对应
public void init(JSONArray data) {
	for (int i = 0; i < data.size(); i++) {
		name = new %ClassName%[data.size()];
		name = JSONObject.parseObject(data.getJSONObject(i).toJSONString(), %ClassName%.class);
	}
}
```		
