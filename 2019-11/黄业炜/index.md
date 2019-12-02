### 1. SQL Server 2012特性分页

1. 以前分页的时候，我基本都是用ROW_NUMBER()函数，每次都得写子查询。
2. 在SQL Server 2012中就不需要写子查询啦，直接在ORDER BY语句中就可以实现分页。
3. OFFSET指定在返回查询结果之前要跳过的行数，FETCH指定OFFSET之后返回的行数。
   
**OFFSET：向SELECT查询表示跳过多少行！**

**FETCH：表示从特定的位置开始检索检索多少行！**

 #### 关键代码如下：  

```csharp
SELECT * FROM [dbo].[WfFlow](NOLOCK)
			ORDER BY [FlowID] asc
			OFFSET 10 ROWS
			FETCH NEXT 10 ROWS ONLY;

```

### 2.Swagger 给每个请求默认加上 Authorization 验证
1. 这个功能主要是BaseController上打了Authorize标签，每个请求都要带上Token。在Swagger上就不能直接调用，要复制参数到PostMan上请求。有这个标签就可以不用复制参数到PostMan上请求，直接在Swagger页面上就可以测试接口。
2. AuthTokenHeaderParameter 继承 IOperationFilter实现Apply方法 例：
 
```csharp
 public class AuthTokenHeaderParameter : IOperationFilter
    {
        public void Apply(Operation operation, OperationFilterContext context)
        {
            operation.Parameters = operation.Parameters ?? new List<IParameter>();
            //MemberAuthorizeAttribute 自定义的身份验证特性标记
            var isAuthor = operation != null && context != null;
            if (isAuthor)
            {
                //in query header 
                operation.Parameters.Add(new NonBodyParameter()
                {
                    Name = "Authorization",
                    In = "header", //query formData ..
                    Description = "身份验证",
                    Required = false,
                    Type = "请输入Token，格式为Bearer XXX"
                });
            }
        }
    }

```
### 3. NET Core 使用swagger进行分组显示
1. 在StarUp.cs的ConfigureServices中添加代码:

```csharp
//注册Swagger生成器，定义一个和多个Swagger 文档
 return services.AddSwaggerGen(c =>
            {
                SpecificationList.ForEach(x =>
                {
                    c.SwaggerDoc(x.Key,
                   new Info
                   {
                       Title = x.Name,
                   });
                });
                //设置要展示的接口
                c.DocInclusionPredicate((docName, apiDesc) =>
                {
                    if (!apiDesc.TryGetMethodInfo(out MethodInfo methonInfo)) return false;
                    var versions = methonInfo.DeclaringType
                    .GetCustomAttributes(true)
                    .OfType<ApiExplorerSettingsAttribute>()
                    .Select(attr => attr.GroupName);

                    if (docName.ToLower() == SpecificationList[1].Key && versions.FirstOrDefault() == null)
                    {
                        return true;
                    }
                    return versions.Any(x => x.ToString() == docName);
                });

                apiCollection.ForEach(x =>
                {
                    var xmlFile = $"{x.Name}.xml";
                    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
                    if (File.Exists(xmlPath))
                        c.IncludeXmlComments(xmlPath);

                    if (x.References != null)
                    {
                        x.References.ForEach(k =>
                        {
                            xmlFile = $"{k.Name}.xml";
                            xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
                            if (File.Exists(xmlPath))
                                c.IncludeXmlComments(xmlPath);
                        });
                    }
                });
                //配置AuthToken验证
                c.OperationFilter<AuthTokenHeaderParameter>();
            });
```

2. 在StarUp.cs的Configure中添加代码:

```csharp
 var SpecificationList = Configuration.GetSection(ConfigKey.Specification).Get<List<Specification>>();
            app.UseSwagger();
            app.UseSwaggerUI(c =>
            {
                SpecificationList.ForEach(x =>
                {
                    string url = string.Format("/swagger/{0}/swagger.json", x.Key);
                    c.SwaggerEndpoint(url, x.Name);
                });
                c.RoutePrefix = string.Empty;
            }); 
```
3. 在controller或者action上打上

```csharp
[ApiExplorerSettings(GroupName = "v1")]
```

4. 在appsettings上配置Swagger Doc版本

```csharp
"specification": [
      {
        "key": "v1",
        "name": "框架API"
      },
      {
        "key": "v2",
        "name": "业务API"
      }
    ],
``` 
   