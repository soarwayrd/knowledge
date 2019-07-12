1. **.NET Core中如何限制上传附件的大小？.NET Core中RequestBody的默认限制大小为30,000,000 bytes = 28.6MB，修改默认值有两种方案：**
   - 第一种，接口添加属性标签RequestSizeLimit，如下代码段：
   ``` cshartp
    [RequestSizeLimit(100000000)]//单位bytes
    public IActionResult Post(IList<IFormFile> files)
    {
        
    }
   ```
   - 通过全局配置修改限制，在Program.cs文件中修改，如下代码段：
    ``` cshartp
    public static IWebHostBuilder CreateWebHostBuilder(string[] args)
    {
        return Core.API.Program.CreateBaseWebHostBuilder(args)
                            .UseStartup<Startup>()
                            //限制HTTP请求主体的最大尺寸
                            .UseKestrel(options => { options.Limits.MaxRequestBodySize = AppSettings.Get<int>(ConfigKey.MaxRequestBodySize) * 1024; });
    }
   ```
2. **JS利用Blob实现文件分片上传，如下关键代码段：**
   > 实现步骤：前端利用 `blob.slice` 实现文件分片 -> 调用服务端接口将所有分片上传 -> 调用接口进行分片合并
   
    ``` javascript
    <div class="jumbotron">
        <h1>文件分片上传简单实例</h1>
    </div>

    <div class="row">
        <div class="col-md-4">
            <h4>上传文件</h4>
            <p><input type="file" name="file" id="file" /></p>
            <p><a class="btn btn-primary" href="#" onclick="upload()">上传 &raquo;</a></p>
        </div>
        <div class="col-md-8">
            <h4>技术应用</h4>
            <p>借助js的Blob对象FormData对象可以实现大文件分片上传的功能</p>
        </div>
    </div>
    <script type="text/javascript">
        var bytesPerPiece = 2 * 1024 * 1024; // 每个文件切片大小定为2MB .
        var totalPieces;

        //发送请求
        function upload() {
            if (document.getElementById("file").files.length == 0) {
                alert('请选择要上传的文件！');
                return;
            }
            var blob = document.getElementById("file").files[0];
            var blobSlice = blob.mozSlice || blob.webkitSlice || blob.slice;
            var start = 0;
            var end;
            var index = 0;
            var filesize = blob.size;
            var filename = blob.name;
            var id = uuid(32, 16);
            var result = true;

            //计算文件切片总数
            totalPieces = Math.ceil(filesize / bytesPerPiece);
            while (start < filesize) {
                end = start + bytesPerPiece;
                if (end > filesize) {
                    end = filesize;
                }

                var chunk = blob.slice(start, end);//切割文件
                var formData = new FormData();
                formData.append("file", chunk, filename);
                formData.append("id", id);
                if (bytesPerPiece <= filesize) {
                    formData.append("chunk", index);
                    formData.append("chunks", totalPieces);
                }
                $.ajax({
                    url: 'Upload',
                    type: 'POST',
                    cache: false,
                    async: false,
                    data: formData,
                    processData: false,
                    contentType: false,
                    success: function (res) {
                        if (res.IsSuccess) {
                            if (bytesPerPiece <= filesize && index == (totalPieces - 1))
                                mergeFiles(id, res.ext);
                        }
                        else {
                            alert(res.Error);
                            result = false;
                        }
                    },
                    error: function (e) {
                        result = false;
                    }
                })
                if (!result)
                    break;

                start = end;
                index++;
            }
        }

        function mergeFiles(id, ext) {
            $.ajax({
                url: 'MergeFiles',
                type: 'POST',
                cache: false,
                async: false,
                data: { id: id, ext: ext },
                success: function (res) {
                    if (res.IsSuccess) {
                        alert("文件上传成功！");
                    }
                    else {
                        alert(res.Error);
                    }
                },
                error: function (e) {

                }
            })
        }

        function uuid(len, radix) {
            var chars = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'.split('');
            var uuid = [], i;
            radix = radix || chars.length;

            if (len) {
                for (i = 0; i < len; i++) uuid[i] = chars[0 | Math.random() * radix];
            } else {
                var r;
                uuid[8] = uuid[13] = uuid[18] = uuid[23] = '-';
                uuid[14] = '4';

                for (i = 0; i < 36; i++) {
                    if (!uuid[i]) {
                        r = 0 | Math.random() * 16;
                        uuid[i] = chars[(i == 19) ? (r & 0x3) | 0x8 : r];
                    }
                }
            }
            return uuid.join('');
        }
    </script>
   ```
   ``` csharp
   public class UploadFilesController : Controller
    {
    /// <summary>
    /// 文件存放路径
    /// </summary>
    const string UploadFiles = "/UploadFiles/";

    // GET: UploadFiles
    public ActionResult Index()
    {
    return View();
    }

    public ActionResult Upload(HttpPostedFileBase file)
    {
    try
    {
    string root = Server.MapPath(UploadFiles);
    string key = Request["id"];
    //如果进行了分片
    if (Request.Form.AllKeys.Any(m => m == "chunk"))
    {
        //取得chunk和chunks
        int chunk = Convert.ToInt32(Request.Form["chunk"]);//当前分片在上传分片中的顺序（从0开始）
        int chunks = Convert.ToInt32(Request.Form["chunks"]);//总分片数
                                                            //根据GUID创建用该GUID命名的临时文件夹
        string folder = root + "chunk\\" + key + "\\";
        string path = folder + chunk;

        //建立临时传输文件夹
        if (!Directory.Exists(Path.GetDirectoryName(folder)))
        {
            Directory.CreateDirectory(folder);
        }

        FileStream addFile = null;
        BinaryWriter AddWriter = null;
        Stream stream = null;
        BinaryReader TempReader = null;

        try
        {
            addFile = new FileStream(path, FileMode.Create, FileAccess.Write);
            AddWriter = new BinaryWriter(addFile);
            //获得上传的分片数据流
            stream = file.InputStream;
            TempReader = new BinaryReader(stream);
            //将上传的分片追加到临时文件末尾
            AddWriter.Write(TempReader.ReadBytes((int)stream.Length));
        }
        finally
        {
            if (addFile != null)
            {
                addFile.Close();
                addFile.Dispose();
            }
            if (AddWriter != null)
            {
                AddWriter.Close();
                AddWriter.Dispose();
            }
            if (stream != null)
            {
                stream.Close();
                stream.Dispose();
            }
            if (TempReader != null)
            {
                TempReader.Close();
                TempReader.Dispose();
            }
        }

        return Json(new { IsSuccess = true, ext = Path.GetExtension(Request.Files[0].FileName) });
    }
    else//没有分片直接保存
    {
        string path = root + key + Path.GetExtension(Request.Files[0].FileName);
        file.SaveAs(path);

        return Json(new { IsSuccess = true, ext = Path.GetExtension(Request.Files[0].FileName) });
    }
    }
    catch(Exception ex)
    {
    return Json(new { IsSuccess = false, Error = ex.Message });
    }
    }

    #region 合并文件
    /// <summary>
    /// 合并文件
    /// </summary>
    /// <returns></returns>
    public ActionResult MergeFiles()
    {
        try
        {
            string root = Server.MapPath(UploadFiles);

            string guid = Request["id"];
            string ext = Request["ext"];
            string sourcePath = Path.Combine(root, "chunk\\" + guid + "\\");//源数据文件夹
            string targetPath = Path.Combine(root, guid + ext);//合并后的文件

            DirectoryInfo dicInfo = new DirectoryInfo(sourcePath);
            if (Directory.Exists(Path.GetDirectoryName(sourcePath)))
            {
                FileInfo[] files = dicInfo.GetFiles();
                foreach (FileInfo file in files.OrderBy(f => int.Parse(f.Name)))
                {
                    FileStream addFile = new FileStream(targetPath, FileMode.Append, FileAccess.Write);
                    BinaryWriter AddWriter = new BinaryWriter(addFile);

                    //获得上传的分片数据流 
                    Stream stream = file.Open(FileMode.Open);
                    BinaryReader TempReader = new BinaryReader(stream);
                    //将上传的分片追加到临时文件末尾
                    AddWriter.Write(TempReader.ReadBytes((int)stream.Length));
                    //关闭BinaryReader文件阅读器
                    TempReader.Close();
                    stream.Close();
                    AddWriter.Close();
                    addFile.Close();

                    TempReader.Dispose();
                    stream.Dispose();
                    AddWriter.Dispose();
                    addFile.Dispose();
                }
                DeleteFolder(sourcePath);

                return Json(new { IsSuccess = true });
            }
            else
            {
                return Json(new { IsSuccess = false, Error = "未找到指定文件！" });
            }
        }
        catch(Exception ex)
        {
            return Json(new { IsSuccess = false, Error = ex.Message });
        }
    }
    #endregion

    #region 删除文件夹及其内容
    /// <summary>
    /// 删除文件夹及其内容
    /// </summary>
    /// <param name="dir"></param>
    private void DeleteFolder(string strPath)
    {
    //删除这个目录下的所有子目录
    if (Directory.GetDirectories(strPath).Length > 0)
    {
    foreach (string fl in Directory.GetDirectories(strPath))
    {
        Directory.Delete(fl, true);
    }
    }
    //删除这个目录下的所有文件
    if (Directory.GetFiles(strPath).Length > 0)
    {
    foreach (string f in Directory.GetFiles(strPath))
    {
        System.IO.File.Delete(f);
    }
    }
    Directory.Delete(strPath, true);
    }
    #endregion
    }
   ```
3. **文件分片上传之断点续传**
    
    `解决问题`：在进行分片上传时由于网络原因或者刷新了界面导致文件仅有一部分的片段上传到了服务器，那么下次再传的时候如果想做到已经上传的不重复传应该如何做呢？

    `核心思想`：如果我们想要续传，那么我们必须知道此次上传的文件是否已经上传了一部分，但由于我们是分片上传的，导致无法识别出原来上传的片段是否属于当前文件，所以我们需要给文件一个唯一的标识，把文件的片段全部存放在以标识命名的文件夹下，这样就可以清楚的知道哪些片段属于哪些文件，也就可以跳过那些已经上传好的片段，这里我们需要通过，核心代码如下：
    ``` javascript
    //利用FileReader和SparkMD5实现分片段对文件进行加密处理
    this.loadFromBlob = function (file, progress, callback) {
        var blob = file.source,
            chunkSize = 2 * 1024 * 1024,
            chunks = Math.ceil(blob.size / chunkSize),
            chunk = 0,
            spark = new SparkMD5.ArrayBuffer(),
            blobSlice = blob.mozSlice || blob.webkitSlice || blob.slice,
            loadNext, fr;

        fr = new FileReader();

        loadNext = function () {
            var start, end;

            start = chunk * chunkSize;
            end = Math.min(start + chunkSize, blob.size);
            //文件读取成功完成时触发
            fr.onload = function (e) {
                spark.append(e.target.result);
            };
            //读取完成触发，无论成功或失败
            fr.onloadend = function () {
                fr.onloadend = fr.onload = null;
                if (progress) {
                    var percent = (chunk + 1) / chunks;
                    progress(percent, file);
                }
                if (++chunk < chunks) {
                    setTimeout(loadNext, 1);
                } else {
                    setTimeout(function () {
                        var result = spark.end();
                        if (callback)
                            callback(result, file);
                        loadNext = file = blob = spark = null;
                    }, 50);
                }
            };
            //按字节读取文件内容，并转换为ArrayBuffer对象（二进制缓冲区）
            fr.readAsArrayBuffer(blobSlice.call(blob, start, end));
        };

        loadNext();
    }
    ```
4. **Jenkins使用说明文档**

    > 文档参考地址：http://192.168.0.225:5656/jenkins/