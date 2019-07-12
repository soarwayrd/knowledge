## url请求参数配置 ##

**http请求中header参数Value值不能是中文，要urlencode编码。**

## net core 部署在IIS ##

默认生成的是**AspNetCoreModuleV2**要把v2删了
``` csharp
<handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
</handlers>
```
出现的错误：Application is running inside IIS process but is not configured to use IIS server.

## git 取消已跟踪的文件 ##

在gitignore文件配置：**/Migrations**

**git rm -r --cached Soarway.Hummer.SmartBuilding.Repository/Migrations**
