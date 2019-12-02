### 1. Ef Core下添加静态及动态全局过滤器
1. 静态全局过滤器
   比如数据表有个IsDelete、Enable字段，其值是固定的，不是为True就是为Flase。
  
   这种的全局过滤器，直接定义在DbSet的Configuration内即可。

   以下代码表示，在做全表查询的时候，会自动加上Where IsDelete = true的条件
   
   ```csharp
   public class InvoiceHistoryConfiguration : IEntityTypeConfiguration<InvoiceHistory>
    {
        public void Configure(EntityTypeBuilder<InvoiceHistory> builder)
        {
            //table
            builder.ToTable($"{typeof(InvoiceHistory).Name}");

            // key
            builder.HasKey(p => p.InvoiceHistoryID);

            // properties
            
            // ingore code 

            // clause
            builder.HasQueryFilter(x => x.IsDelete == false);
        }
    }
   ```
2. 动态全局过滤器

    以SaaS系统为例，多个租户共享一个数据库，并通过TenantID来"软区隔"出不同用户之间的数据。也就是说，每张数据表里都会有TenantID这个字段。

    倘若这时候有一条SQL语句忘记在结尾处加上Where TenantID = @Tenant, 那后果是很严重的，将会导致其他租户的敏感数据泄露。且这种逻辑上的BUG相对其他功能上的BUG来的更为隐蔽，且危害更大。

    这时候，用全局过滤器就能很好的解决这个问题。但是每张数据表都有TenantID, 我们当然可以为每一个DbSet的Configuration配置HasQueryFilter函数。但是，可以通过技术手段(反射)和约定(接口)，在统一的一个地方进行添加和管理。

    ```csharp

        public DbContextBiz(IHttpContextAccessor httpContextAccessor) : base(httpContextAccessor)
        {
        }

        protected void SetDefaultQueryFilter<T>(ModelBuilder modelBuilder, string methodName) where T : class
        {
            foreach (var type in modelBuilder.Model.GetEntityTypes())
            {
                if (type.ClrType.IsSubclassOf(typeof(T)))
                    SetDefaultQueryFilter(modelBuilder, type.ClrType, methodName);
            }
        }

        private void SetDefaultQueryFilter(ModelBuilder modelBuilder, Type entityType, string methodName)
        {
            MethodInfo SetSoftDeleteFilterMethod = typeof(DbContextBiz).GetMethod(methodName, 1, new Type[] { typeof(ModelBuilder) });
            SetSoftDeleteFilterMethod.MakeGenericMethod(entityType).Invoke(this, new object[] { modelBuilder });
        }

        public void BaseInvoiceQueryFilter<T>(ModelBuilder modelBuilder) where T : BaseInvoice
        {
            string nickName = _httpContextAccessor.HttpContext.GetNickName();
            string openID = _httpContextAccessor.HttpContext.GetOpenID();

            Expression<Func<T, bool>> expression = t => t.IsDelete == false;

            if (!string.IsNullOrWhiteSpace(nickName))
                expression = expression.And(t => t.NickName == nickName);

            if (!string.IsNullOrWhiteSpace(openID))
                expression = expression.And(t => t.OpenID == openID);

            modelBuilder.Entity<T>().HasQueryFilter(expression);
        }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
            if (_httpContextAccessor != null)
                SetDefaultQueryFilter<BaseInvoice>(modelBuilder, "BaseInvoiceQueryFilter");
        }

    ```

    ```csharp
     public class BaseInvoice
     {
 
         /// <summary>
         /// 关注公众号的用户的微信OpenID
         /// </summary>
         [Required]
         public string OpenID { get; set; }
 
         /// <summary>
         /// 开票用户微信昵称
         /// </summary>
         [Required]
         public string NickName { get; set; }
 
         /// <summary>
         /// 是否删除
         /// </summary>
         public bool IsDelete { get; set; } = false;
     }
    ```

3. 注意事项
   > HasQueryFilter函数只能调用一次，如果调用多次，只有最后一次的调用才生效

   > 运行过程中，如果需要取消全局过滤器，可以调用IQueryable.IgnoreQueryFilters()方法进行忽略，单次有效。