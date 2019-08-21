#### 使用RazorLight生成HTML内容

1. 使用的插件是 RazorLight，安装RazorLight 有一个地方需要注意就是在NuGet程序包上搜索找到的只有1.1.0版本。这是这个版本我们用不了。必须使用NuGet 包管理器->程序包管理器控制台安装2.0.0-beta1，目前只有beta1版本的。
   
```javascript 
    Install-Package RazorLight -Version 2.0.0-beta1
```
2. 通过Model生成HTML只要调用
   
```csharp

    data model = new data() { Name = "Jack", Age = 18, NickName = "Test" };
    string str = "Hello, @Model.Name. Welcome to RazorLight repository.Age=@Model.Age . NickName=@Model.NickName";

    //result返回的是一个生成好的HTML内容
    string result = RenderService.RazorRender<data>(str, model);
    //Hello, Jack. Welcome to RazorLight repository.Age=18 . NickName=Test

```

  - [文档地址](https://github.com/toddams/RazorLight#enable-intellisense-support)

1. 使用Aspose.Words 生成PDF、DOCX等文件。Aspose下载的net core版本。添加引用到项目里
   >根据模板生成文件保存到FTP

   ```csharp
     bool bol = false;
    //生成HTML模板
    string result = RenderService.RazorRender<data>(FilePath, model);
    if (!string.IsNullOrEmpty(result))
        //文件保存到FTP
        bol = RenderService.SaveFTPFile(result, SavePath, 2, SaveFormat.Pdf);   
   ```
   >根据模板生成文件返回Stream

    ```csharp
    Stream stream = null;
    //生成HTML模板
    string html = RenderService.RazorRender<data>(FilePath, model);
    if (!string.IsNullOrEmpty(html))
    {
        //文件保存到FTP
        bol = RenderService.SaveFTPFile(result, SavePath, 2, SaveFormat.Pdf);
        if (bol)
            //返回文件流
            stream = FTPHelper.FileToStream(SavePath);
    }
    ```
    >文件保存到本地

    ```csharp
     html = RenderService.RazorRender<data>(FilePath, model);
     if (!string.IsNullOrEmpty(html))
        RenderService.GenerateFile(html, SavePath, SaveFormat.Pdf);
    ```