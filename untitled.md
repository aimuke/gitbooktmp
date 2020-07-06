# Untitled





 **四、用法演示：**  
1、下载页面：

| 1 | `curl -o index.html http://aiezu.com` |
| :--- | :--- |


   
2、下载文件并显示简单进度条：

| 1 | `curl -# -o centos6.8.iso http://mirrors.aliyun.com/centos/6.8/isos/x86_64/CentOS-6.8-x86_64-minimal.iso` |
| :--- | :--- |


   
3、断点续传：

| 12 | `#继续完成上次终止的未完成的下载curl -# -o centos6.8.iso -C - http://mirrors.aliyun.com/centos/6.8/isos/x86_64/CentOS-6.8-x86_64-minimal.iso` |
| :--- | :--- |


  
4、伪造来源页面：  


| 12 | `#告诉爱E族，我是从百度来的curl -e http://baidu.com http://aiezu.com` |
| :--- | :--- |


  
 5、伪造代理设备：

| 12345 | `#告诉爱E族，我是GOOGLE爬虫蜘蛛（其实我是curl命令）curl -A " Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"` `http://aiezu.com` `#告诉爱E族，我用的是微信内置浏览器curl -A "Mozilla/5.0 AppleWebKit/600 Mobile MicroMessenger/6.0"` `http://aiezu.com` |
| :--- | :--- |


   
6、http头：

| 12 | `# 看看本站的http头是怎么样的curl -I  http://aiezu.com` |
| :--- | :--- |


输出：

| 123456789 | `HTTP/1.1 200 OKDate: Fri, 25 Nov 2016 16:45:49 GMTServer: ApacheSet-Cookie: rox__Session=abdrt8vesprhnpc3f63p1df7j4; path=/Expires: Thu, 19 Nov 1981 08:52:00 GMTCache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0Pragma: no-cacheVary: Accept-EncodingContent-Type: text/html; charset=utf-8` |
| :--- | :--- |


   
6、设置http请求头：

| 1 | `curl -H "Cache-Control:no-cache"`  `http://aiezu.com` |
| :--- | :--- |


  
7、发送表单数据：

| 1 | `curl -F "pic=@logo.png"` `-F "site=aiezu"`  `http://aiezu.com/` |
| :--- | :--- |


  
8、发送cookie：

| 1 | `curl -b "domain=aiezu.com"`  `http://aiezu.com` |
| :--- | :--- |


