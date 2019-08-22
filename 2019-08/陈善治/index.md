1. Gitbook嵌入代码的BUG
在md文档内潜入代码块的时候，通过\```就可以标识这个区块为代码片段。
当有的时候，在Visual Studio Code内预览代码片段都正常，但是Gitbook编译后，却发生了错位。如果发生这种情况，则可以观察一下你的代码与\```之间的位置关系，代码最好要排在\```之后。

2. EntityFrameworkCore默认值的问题
    [官方解答](https://github.com/aspnet/EntityFrameworkCore/issues/15070)

    在使用EntityFrameworkCore时，请慎重使用字段默认值。

    ```csharp
    public class User 
    {
        /// <summary>
        /// 用户性别 男:true, 女:false 
        /// </summary>
        public bool Gender { get; set; }
    }

    public class UserConfiguration : IEntityTypeConfiguration<User>
    {
        public virtual void Configure(EntityTypeBuilder<User> builder)
        {
            //properties
            builder.Property(p => p.Gender).HasDefaultValue(true).IsRequired(true);
        }
    }
    ```

    一旦按照如上代码定义字段默认值后，某些情况下，代码的执行结果会出乎你所料。
    比如在Ado.Net的时代，我们也经常定义数据库字段的默认值。以**Gender**为例，当我向数据库插入数据时，如果我不传**Gender**字段，那么数据库将会填入定义的默认值，但是如果我传了**Gender**的值，这数据库将会放弃默认值，选用我传的值，无论True/False。

    但是在EntityFrameworkCore下，这一切都变了。只要你定义了的默认值，不是该类型的默认值，那么执行的结果就将不会按你预想的那样。在EntityFrameworkCore下，还以**Gender**为例，我设置了默认值为True，我不传值的时候，选用默认值True；但当我传的值为False时候，插入的数据依然为True。通过Sql Profile追踪可以发现，一旦设置了bool类型的默认值后，Ef Core生成的Insert语句里就不在体现该字段了，而是直接去数据库的默认值。Ef Core的这种设计思路与传统的Ado.Net的思路完全不一样。为了避免不必要的麻烦，在Ef Core里使用默认值的时候要明记这个很特殊的地方。


3. EntityFrameworkCore的导航属性的问题

    集合类型的导航属性，**必须实例化**；单体对象的导航属性，一定**不能实例化**，只需定义即可。

    >集合对象：如果不实例化，但导航属性为空的时候，其值就为null，如果实例化后，这只是Count为0；

    >单体对象：如果实例化，这会在新增的时候，在单体对象的导航属性里插入一条空数据，所以保持为null即可。

    ```csharp
    public class UserGroup 
    {
        public int UserGroupID { get; set; }
        public int UserID { get; set; }
        public int GroupID { get; set; }
        // 一定不能实例化
        public virtual Group Group { get; set; }
        // 一定不能实例化
        public virtual User User { get; set; }
    }
    ```

    ```csharp
    public class User 
    {
        public User()
        {
            // 必须实例化
            this.UserGroupRoles = new HashSet<UserGroupRole>();
            // 必须实例化
            this.UserRoles = new HashSet<UserRole>();
            // 必须实例化
            this.UserGroups = new HashSet<UserGroup>();
        }
        
        public virtual ICollection<UserGroupRole> UserGroupRoles { get; set; }
        public virtual ICollection<UserRole> UserRoles { get; set; }
        public virtual ICollection<UserGroup> UserGroups { get; set; }
        public virtual ICollection<ResourceAuth> ResourceAuths { get; set; }
    }
    ```
4. Autofac生命周期管理
    [官方解释](https://autofaccn.readthedocs.io/en/latest/integration/aspnetcore.html#differences-from-asp-net-classic)

    **InstancePerLifetimeScope**与**InstancePerRequest**

    **InstancePerRequest** [官方解释](https://autofac.readthedocs.io/en/latest/lifetime/instance-scope.html#instance-per-lifetime-scope)
    **InstancePerLifetimeScope** [官方解释](https://autofac.readthedocs.io/en/latest/lifetime/instance-scope.html#instance-per-request)

    InstancePerRequest 是给.Net用的，在.Net Core下会报错，InstancePerLifetimeScope是给.Net Core 2.0+ 用的。
    内部实现机制不同，但是实现的效果类似，实现的都是**一个请求一个实例**的效果。