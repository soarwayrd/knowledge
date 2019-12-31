### AsNoTracking()

在Ef Core内，进行数据查询时，默认会将查询的结果进行跟踪。

正如Ef Core做Update操作时，常规动作是先用ID将数据查出来，然后再逐一修改字段只，最后通过SaveChange()写回数据。

```csharp
 public StatusResult<EffectCountDTO> ModifyBook(int id, BookModifyDTO dto)
 {
     StatusResult<EffectCountDTO> statusResult = new StatusResult<EffectCountDTO>();
     var entity = _BookRepository.GetByPrimaryKey(id);
     Mapper.Map<BookModifyDTO, Book>(dto, entity);
     var result = new EffectCountDTO() { EffectCount = _BookRepository.Update(entity) };
     return statusResult.Ok(data: result);
 }
```


总结就是，先查询，在编辑，最后写回数据库。由于Ef Core默认会对Entity进行跟踪，所以在最后写回数据库时，才会知道应该对哪些字段进行修改。

这种方式用起来确实很方便，但是大部分的也许系统，Update（编辑）等操作相对于Select（查询）而言，其频率是很低的。

如果你的函数时纯查询为主的，应告诉Ef Core，你要进行AsNoTracking操作。这样能大幅提高Ef Core的查询性能，节约资源。

使用方式及其友好，只需在IQueryable接口上调用AsNoTracking()即可

```csharp
 public StatusResult<List<BookResultDTO>> GetBookLis (BookQueryDTO dto)
 {
     StatusResult<List<BookResultDTO>> statusResult = new StatusResult<List<BookResultDTO>>();
     var result = _BookRepository.FindBySpecification(t => true)
                                 // 注意此行代码
                                 .AsNoTracking()
                                 .ToList<Book, BookResultDTO>();
     return statusResult.Ok(data: result);
 }
```

一旦加了AsNoTracking，Ef Core就不在对当前查询的结果进行跟踪，在查询结果集上做的任何修改，都不在被作用到数据库上。

> 纯的查询功能，才可以添加AsNoTracking以便于提高性能；如果非纯查询，有可能导致结果不可预期。

### Inlcude() / ThenInclude()

在Ef Core内，我们可以设置导航属性很方便的进行关联数据的获取。

Book内带有一个导航属性指向BookCategory, 在写好Mapper规则的情况下，Book的查询结果会自动导航到BookCategory上，拿到BookCategoryName字段，赋值给BookResultDTO内的同名字段。


```csharp
 public class BookResultDTO
 {
     /// <summary>
     /// 书本名称
     /// </summary>
     public string BookName { get; set; } 
     /// <summary>
     /// 书本ISBN号
     /// </summary>
     public string ISBN { get; set; } 
     /// <summary>
     /// 书本售价
     /// </summary>
     public decimal Price { get; set; } 
     /// <summary>
     /// 出版日期
     /// </summary>
     public string PublishDate { get; set; } 
     /// <summary>
     /// 出版单位
     /// </summary>
     public string PublishOrg { get; set; } 
     /// <summary>
     /// 书本分类名称
     /// </summary>
     public string BookCategoryName { get; set; }
 }
```

```csharp
 public class Book
 {
     /// <summary>
     /// 书本ID
     /// </summary>
     public int BookID { get; set; }

     /// <summary>
     /// 书本名称
     /// </summary>
     public string BookName { get; set; }

     /// <summary>
     /// 书本ISBN号
     /// </summary>
     public string ISBN { get; set; }

     /// <summary>
     /// 书本售价
     /// </summary>
     public decimal Price { get; set; }

     /// <summary>
     /// 出版日期
     /// </summary>
     public string PublishDate { get; set; }

     /// <summary>
     /// 出版单位
     /// </summary>
     public string PublishOrg { get; set; }

     /// <summary>
     /// 书本分类ID
     /// </summary>
     public int BookCategoryID { get; set; }

     /// <summary>
     /// 书本分类
     /// </summary>
     public virtual BookCategory BookCategory { get; set; }
 }
```

```csharp
 public class BookCategory
 {
     /// <summary>
     /// 书本分类ID
     /// </summary>
     public int BookCategoryID { get; set; }
 
     /// <summary>
     /// 书本分类名称
     /// </summary>
     public string CategoryName { get; set; }
 }
```

对比一下的两段代码，对查询结果而言，代码一和代码二等价。

但是如果从Sql Profile追踪来看，缺天壤之别。代码二远优于代码一，

同样以10条数据为例，最粗暴的比较，性能差距在20倍左右：

代码一，先查出代码一的10调数据，在进行Mapper时，对导航属性BookCategory进行查询，10条Book数据，对应的就查询了10次BookCategory，总查询次数为20次。

代码二，直接生成了Book Left Join BookCategory的语句，进行了联合查询，返回与代码一一样的结果，但是总查询次数只有1次。

```csharp
 // 代码一
 public StatusResult<PageResult<BookResultDTO>>  GetBookPage(BookQueryDTO dto)
 {
     StatusResult<PageResult<BookResultDTO>> statusResult = new StatusResult<PageResult<BookResultDTO>>();
     var expression = GetExpression(dto);
     var pageResult = _BookRepository.FindBySpecification(expression)
                                     .ToPageResult<Book, BookResultDTO>(dto.Num, dto.Size);
     return statusResult.Ok(data: pageResult);
 }
```
 
```csharp
 // 代码二
 public StatusResult<PageResult<BookResultDTO>> GetBookPage(BookQueryDTO dto)
 {
     StatusResult<PageResult<BookResultDTO>> statusResult = new StatusResult<PageResult<BookResultDTO>>();
     var expression = GetExpression(dto);
     var pageResult = _BookRepository.FindBySpecification(expression)
                                     // 注意此行代码
                                     .Include(x => x.BookCategory)
                                     .ToPageResult<Book, BookResultDTO>(dto.Num, dto.Size);
     return statusResult.Ok(data: pageResult);
 }
```
