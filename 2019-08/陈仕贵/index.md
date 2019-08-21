# Angular富文本编辑器tinymce-angular #
TinyMCE是一个轻量级的富文本编辑器，相对于CKEditor更加精简，但足以满足绝大部分场景的需要。目前网上关于Angular整合TinceMCE的文章不少，但讲述得不是很清晰，经过一番折腾，小编总算比较完美地将它整合到Angular项目中
## 一、安装TinyMCE ##
[官方网站](https://www.tiny.cloud/) 上有比较完整的angular+tinymce安装指引，所以这部分我就比较快速带过。
### 1、安装tinymce ###
npm install --save tinymce
### 2、设置tinymce全局访问 ###
与安装一般的npm包不同，tinymce需要设置全局访问之后才能够使用，也就是要在项目的.angular-cli.json文件中添加以下内容：
### 3、定义全局变量 ###
你还需要在项目中的.\src\typing.d.ts中声明tinymce全局变量，不然会提示找不到tinymce
declare var tinymce: any;

### 4、拷贝皮肤文件到assets目录下 ###
tinymce的主题（theme）跟皮肤（skin）是相互分离的，皮肤主要是字体、图标、css等一些内容。我们需要将相关文件拷贝到项目中的assets目录下。也就是将.\node_modules\tinymce中的skins目录整个拷贝到.\src\assets目录下。

### 5、安装中文支持 ###
tinymce默认是英文界面，如果要使用中文，我们需要先下载中文语言包，然后将其路径加入到上面的全局配置当中。
中文语言包可以从这个地址下载：[地址](https://www.tiny.cloud/)
下载下来的压缩文件中会有一个langs目录，里面有zh_CN.js，我们可以把这个目录拷贝到.\src\assets目录下，然后在全局中添加引用：

## 二、使用tinymce ##

插件

powerpaste

附件上传

file_picker_callback: function (callback, value, meta) {}

图片上传

images_upload_handler: function (blobInfo, success, failure) {}
