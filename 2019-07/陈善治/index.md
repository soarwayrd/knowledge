# Action<T> 与 Func<T,K>的使用体会

在平时的业务开发中，[Action](https://docs.microsoft.com/en-us/dotnet/api/system.action-1?view=netcore-2.2)和[Func](https://docs.microsoft.com/en-us/dotnet/api/system.func-1?view=netcore-2.2)这两个类，他们均有多个版本的重载。
`Action`和`Func`，曾经一直是被动使用的比较多，主动使用的比较少。但近来在框架的开发过程中，逐渐体会到了巧用这两个类，能让api设计的更精简、灵活。

以在设计仓储基类的一个查询接口为例。

```csharp
// 最终确定的API实现
public virtual IQueryable<T> FindBySpecification(Expression<Func<T, bool>> expression, Func<IQueryable<T>, IQueryable<T>> orderBy)
{
    IQueryable<T> query = Entities.Where(expression);
    query = orderBy.Invoke(query);
    return query;
}

//API调用方式
_userRepository.FindBySpecification(expression,a => a.OrderByDescending(x => x.Password).ThenByDescending(x => x.UserID));
```

接下来再看看前一版API

```csharp
// 第一版API实现
public virtual IQueryable<T> FindBySpecification(Expression<Func<T, ool>> expression, params OrderByDefine[] orderBy)
{
    IQueryable<T> query = Entities.Where(expression);
    orderBy.ToList().ForEach(x => query = query.SuperOrderBy(x.FieldName, x.IsDesc));
    return query;
}

// 动态拼接OrderBy的扩展函数
public static IQueryable<T> SuperOrderBy<T>(this IQueryable<T> source, string rderByProperty, bool desc)
{
    string command = string.Empty;
    if (source.Expression.Type == typeof(IOrderedQueryable<T>))
        command = desc ? "ThenByDescending" : "ThenBy";
    else
        command = desc ? "OrderByDescending" : "OrderBy";
    var type = typeof(T);
    var property = type.GetProperty(orderByProperty);
    var parameter = Expression.Parameter(type, "p");
    var propertyAccess = Expression.MakeMemberAccess(parameter, property);
    var orderByExpression = Expression.Lambda(propertyAccess, parameter);
    var resultExpression = Expression.Call(typeof(Queryable), command, new Type[] { type, property.PropertyType },
                                  source.Expression, Expression.Quote(orderByExpression));
    return source.Provider.CreateQuery<T>(resultExpression);
}

// 排序辅助类的定义
public class OrderByDefine
{
    public OrderByDefine(string fieldName, bool isDesc)
    {
        FieldName = fieldName;
        IsDesc = isDesc;
    }
    public bool IsDesc { get; set; }
    public string FieldName { get; set; }
}

// API调用方式
_userRepository.FindBySpecification(expression,new OrderByDefine("Password", true), new OrderByDefine("UserID", true));
```

总的来说有以下优劣势
1. 参数的强类型的，有编译器支持，编码的过程有智能感知，最大程度的避免了输入错误。
2. 减少的类和相关函数的实现，写得多，错的多，不存在没有BUG的程序。
3. 调用的时候也复杂，要先实例化`OrderByDefine`类，然后在输入相关参数，这个类是自定义类，一定层度使用者上还要去了解这个类。
4. 通过Func<IQueryable<T>, IQueryable<T>>, 把查询语句传给调用者，只需补齐他的OrderBy即可，通过Func的第二个参数，拿回包含了调用者定义的OrderBy。
5. 总的来说，通过Func<T,T>把业务还给调用者，而API方面做得只是`调度`类的工作，将不确定性转移到API外，保持了上层接口稳定。


# .Net项目命名空间的选择

在选用Visual Studio进行.Net或.Net Core项目进行开发时，`namespace`的命名一般是由**Visual Studio**自动创建的。

其创建规则为，以项目名称为`根namespace`，每层文件夹都会成为新的一级`namespace`。尽管**Visual Studio**的本意是好的，但是很多时候`自动`并不等于`方便`。

在项目的开发初期，开发者都会粗略的对不同的功能进行分项目管理，比如接口放在`API`层，工具放在`Common`或者`Unitiy`层，实体放在`Model`层或`DTO`层。

随着项目的开发，项目间逐渐会形成`网状`的应用关系，一不小心，有可能就会出现**项目间循环引用**的问题。

这时候你就要重新审视你的项目分层，把一些类移到合适的层去，让项目间的应用关系清晰，避免**项目间循环引用**的出现。

但是在移类的过程中，常常因为开发者将`namespace`的命名全盘托给**Visual Studio**来管理，造成`namespace`结构深、名字长，此时假设你要移动`ExcelHelper.cs`类移动到一个新的项目下，默认的`ExcelHelper`的`namespace`将会是`Soarway.Demo.Infrastructure.Tools.Export`。

```javascript
├─Soarway.Demo.Infrastructure
  └─Tools
      └─Export
              ExcelHelper.cs
```

将`ExcelHelper`移动到`Soarway.Demo.Utility`项目的如下结构，你需要同步的修改`namespace`为`Soarway.Demo.Utility.Xls.Helper`。

```javascript
└─Soarway.Demo.Utility
    └─Xls
        └─Helper
                ExcelExport.cs
```

但是事情到此，永未完结，所有引用过`ExcelExport`类的地方都需要将`namespace`先做删除，再添加新的`using`。往往项目中，一次移动的会不止一个文件，并且应用这些类的地方也不止一个，你要改的`using`引用会很多。

在.Net Core的框架开发过程中，默认只采用`项目名称`作为所有类的默认的`namespace`,可以很大程度上降低，后期项目的调整的成本。即类放在多深的层级下，其命名空间只用`项目名称`作为`namespace`,以公司现有的项目规模，甚至现有的项目规模翻两倍大小的项目，也不会有什么太大的问题，但由此带来的好处有：

1. 项目结构调整时，成本直线下降。
2. 更能沉浸式开发，IDE智能感知的优势得以最大限度的体现，不用代码写一半，还得跑去加`using`，亦或者要蒙着眼把一个大小写敏感的类名书写正确，然后再添加`using`。
3. 精简了源代码顶端的`using`区域

以下做一个对比

```csharp 
// 易办公使用Smart框架的 UserController类的using区域
using AutoMapper;
using Newtonsoft.Json;
using Soarway.Smart.Common;
using Soarway.Smart.Common.Attributes;
using Soarway.Smart.Common.Output;
using Soarway.Smart.Common.Push.Wechat;
using Soarway.Smart.Common.Push.Wechat.Model;
using Soarway.Smart.Common.Service;
using Soarway.Smart.Common.Utility;
using Soarway.Smart.Common.Utility.JsonModel;
using Soarway.Smart.DAL;
using Soarway.Smart.DTO.CommonDTO;
using Soarway.Smart.DTO.Services;
using Soarway.Smart.DTO.Users;
using Soarway.Smart.Repository;
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Net.Http;
using System.Threading.Tasks;
using System.Web;
using System.Web.Http;
```

```csharp
// .Net Core Hummer框架的UserController 类的using区域
using AutoMapper;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Soarway.Hummer.Core.DbSet;
using Soarway.Hummer.Core.DTO;
using Soarway.Hummer.Core.Infrastructure;
using Soarway.Hummer.Core.IRepository;
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.IO;
using System.Linq;
using System.Linq.Expressions;
using System.Text;
```