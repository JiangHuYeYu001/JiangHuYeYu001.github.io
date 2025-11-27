来源于@ESFJ-MoZhu
使用工具yikit（前端抓包工具，类似于burp）,IDEA（这个不消说）
首先，这是我们需要的key的位置:C:\Users\mozhu\Desktop\FUCKyouASSHOLE\..\..\..\KEY.txt

<img width="933" height="441" alt="Image" src="https://github.com/user-attachments/assets/aef4a305-cdfa-4d22-ac2c-6295d7526f62" />

这个是控制上传文件的函数：

<img width="924" height="990" alt="Image" src="https://github.com/user-attachments/assets/624250fc-dc84-40c7-9303-f9453bdfdddc" />
可以在图中看到：
第一个if只对/进行了检查，但是windows下可以用\
不过我们要发现，第一个箭头指向的地方有个正则检测，

<img width="723" height="222" alt="Image" src="https://github.com/user-attachments/assets/d924e358-81cc-41df-bb38-4fd9cb0fa0e1" />

这个一般比较讨厌。不过毕竟是嵌套在大if下的小if，

<img width="723" height="222" alt="Image" src="https://github.com/user-attachments/assets/7e704c09-2aa7-4bc7-90bf-36e54d9c57a9" />

<img width="546" height="150" alt="Image" src="https://github.com/user-attachments/assets/14aa4d41-7348-4efc-ac5d-19eaf01e88b8" />
这里直接绕过。用反斜杠/即可。

<img width="1107" height="846" alt="Image" src="https://github.com/user-attachments/assets/fb631d80-cd38-4897-9081-f0e44588ac78" />
这里运用了ai的代码审查工具，很有意思

