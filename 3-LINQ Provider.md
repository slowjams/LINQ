## LINQ Provider

===============================================

## Demystifying `IQueryable<T>`

```C#
//----------------------------V
public interface IQueryable<T> : IEnumerable<T>, IQueryable { }

public interface IOrderedQueryable<T> : IQueryable<T>, IOrderedQueryable { }  // public interface IOrderedQueryable : IQueryable { }

public interface IQueryable : IEnumerable 
{
   Type ElementType { get; }

   Expression Expression { get; }

   IQueryProvider Provider { get; }
}

public interface IQueryProvider
{
   IQueryable CreateQuery(Expression expression);

   IQueryable<TElement> CreateQuery<TElement>(Expression expression);

   object Execute(Expression expression);

   TResult Execute<TResult>(Expression expression);
}
//----------------------------Ʌ

//---------------------------V
public static class Queryable  // Enumerable's counterpart, Queryable has the same methods as Enumerable's, but the predicate argument is an Expression, not a delegate
{
   public static IQueryable<TElement> AsQueryable<TElement>(this IEnumerable<TElement> source)
   {
      return new EnumerableQuery<TElement>(source);  // return source as IQueryable<TElement> ?? new EnumerableQuery<TElement>(source);  
   }

   public static IQueryable<TSource> Where<TSource>(this IQueryable<TSource> source, Expression<Func<TSource, bool>> predicate)
   {
      return source.Provider.CreateQuery<TSource>(
         Expression.Call(
            null,
            //get the MethodInfo of new Func<IQueryable<object>, Expression<Func<object, bool>>, IQueryable<object>>(Queryable.Where)
            CachedReflectionInfo.Where_TSource_2(typeof(TSource)), 
            source.Expression,             // ConstExpression that wraps EnumerableQuery<T> (source itself)  AS arg0                               
            Expression.Quote(predicate)    // UnaryExpression that wraps the predicate                       AS arg1
            ));
   }
}
//---------------------------Ʌ

//-----------------------------------VV
public abstract class EnumerableQuery 
{
   internal abstract Expression Expression { get; }
   internal abstract IEnumerable Enumerable { get; }

   internal EnumerableQuery() { }

   internal static IQueryable Create(Type elementType, IEnumerable sequence)
   {
      Type seqType = typeof(EnumerableQuery<>).MakeGenericType(elementType);
      return (IQueryable)Activator.CreateInstance(seqType, sequence);
   }

   internal static IQueryable Create(Type elementType, Expression expression)  //  create an IQueryable instance that can evaluate the query tree
   {
      Type seqType = typeof(EnumerableQuery<>).MakeGenericType(elementType);
      return (IQueryable)Activator.CreateInstance(seqType, expression)!;       // this newly created EnumerableQuery<T> instance's _enumerable is null
   }
}

public class EnumerableQuery<T> : EnumerableQuery, IOrderedQueryable<T>, IQueryProvider  // it is both of IQueryable and IQueryProvider, while the counterpart in the Linq to SQL 
{                                                                                        // in the next section, IQueryable and IQueryProvider are separated
   private readonly Expression _expression;
   private IEnumerable<T> _enumerable;

   public EnumerableQuery(Expression expression) { _expression = expression; }

   public EnumerableQuery(IEnumerable<T> enumerable)
   {
      _enumerable = enumerable;
      _expression = Expression.Constant(this);
   }

   internal override Expression Expression => _expression;
   internal override IEnumerable Enumerable => _enumerable;

   IQueryProvider IQueryable.Provider => this;  // <----------------

   Type IQueryable.ElementType => typeof(T);

   Expression IQueryable.Expression => _expression;

   IQueryable IQueryProvider.CreateQuery(Expression expression)
   {
      Type iqType = TypeHelper.FindGenericType(typeof(IQueryable<>), expression.Type);
      return Create(iqType.GetGenericArguments()[0], expression);
   }

   IQueryable<TElement> IQueryProvider.CreateQuery<TElement>(Expression expression)
   {
      return new EnumerableQuery<TElement>(expression);   // for Where(), expression is MethodCallExpression whose arg0 is ConstExpression that wraps source itself, 
                                                          // arg1 is UnaryExpression that wraps the predicate
   }

   object IQueryProvider.Execute(Expression expression)
   {
      return EnumerableExecutor.Create(expression).ExecuteBoxed();
   }

   TElement IQueryProvider.Execute<TElement>(Expression expression)
   {
      return new EnumerableExecutor<TElement>(expression).Execute();
   }

   IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
   IEnumerator<T> IEnumerable<T>.GetEnumerator() => GetEnumerator();

   private IEnumerator<T> GetEnumerator()
   {
      if (_enumerable == null)   // optimized when you call AsQueryable() but don't call Where() etc, so it is still IEnumerable<T>
      {
         // EnumerableRewriter unwraps the first EnumerableQuery<T> (contains IEnumerable<T>) into a ConstantExpression and also unwrap the UnaryExpression (generated by 
         // Expression.Quote), so don't worry about EnumerableRewriter source code, it just does some unimportant unwrapping
         EnumerableRewriter rewriter = new EnumerableRewriter();  
         Expression body = rewriter.Visit(_expression);   // both _expression and body (some nodes are unwrapped ) are MethodCallExpression        

         // change Where(Const, x => filter) to () => Where(Const, x => filter) 
         Expression<Func<IEnumerable<T>>> f = Expression.Lambda<Func<IEnumerable<T>>>(body, (IEnumerable<ParameterExpression>?)null);  // <--------------------- C2
         IEnumerable<T> enumerable = f.Compile()();
         //
         if (enumerable == this)
            throw Error.EnumeratingNullEnumerableExpression();
         _enumerable = enumerable;
      }
      return _enumerable.GetEnumerator();
   }

   /*
   note that the two statement in C2 are very important, it applys the delegate (represented by the Expression) to the IEnumerable<T> source, it allows you dynamically
   create Expression according to users input at run time, this is exactly what we want in the Process filting example
   */
}
//-----------------------------------ɅɅ

//--------------------------------------VV
public abstract class EnumerableExecutor
{
   internal abstract object? ExecuteBoxed();

   internal EnumerableExecutor() { }

   internal static EnumerableExecutor Create(Expression expression)
   {
      Type execType = typeof(EnumerableExecutor<>).MakeGenericType(expression.Type);
      return (EnumerableExecutor)Activator.CreateInstance(execType, expression)!;
   }
}

public class EnumerableExecutor<T> : EnumerableExecutor
{
   private readonly Expression _expression;

   public EnumerableExecutor(Expression expression) => _expression = expression;

   internal override object? ExecuteBoxed() => Execute();

   internal T Execute()
   {
      EnumerableRewriter rewriter = new EnumerableRewriter();
      Expression body = rewriter.Visit(_expression);
      Expression<Func<T>> f = Expression.Lambda<Func<T>>(body, (IEnumerable<ParameterExpression>?)null);
      Func<T> func = f.Compile();
      return func();
   }
}
//--------------------------------------ɅɅ
```

compare the LINQ built-in, IQueryable, provider with our custom LINQ to SQL Provider that covered in the next section:

`EnumerableQuery<T>` is both the default `IQueryable<T>` and the default `IQueryProvider`. The usage of it is limited to apply the delegate (represented by `Expression`) to the `IEnumerable<T>` souce, so that you can capture users' selection at run time and then apply the logic, in the case, the `IEnumerable<T>` souce is still on client's side, that's what we have done for the ProcessFilter.  In scenarios that the source is on server's side and  we need a custom provider e.g LINQ to SQL, `IQueryable<T>` and `IQueryProvider` are seperated.

That's why `EnumerableQuery<T>`'s `GetEnumerator()` method doesn't use `Execute` method, while `Query<T>`'s `GetEnumerator()` method does call `DbQueryProvider` provider's `Execute` method to return another `IEnumerable<T>` source, in our case, it is a "Database Reader" which implements `IEnumerable<T>`, so when you are enumerating the source, you are actually enumerating another `IEnumerable<T>` created by `IQueryProvider.Execute(Expression expression)` method. So you can think that `IQueryProvider.Execute()` method is to create another `IEnumerable<T>` (e.g `ObjectReader<T>`) and inside `IQueryProvider.Execute()` method, the delegate logic represented by the `Expression` (actually it is `methodCallExpression`) will be applied on the source on server side, in our LINQ to SQL Provider, there is also another class `QueryTranslator` (a `ExpressionVisitor` ) that visits all the node to generate the SQL statement:

Note that in C7,  having an explicit execute instead of just relying on IEnumerable.GetEnumerator() is important because it allows execution of expressions that do not necessarily yield sequences. For example, the query "myquery.Count()" returns a single integer. The expression tree for this query is a method call to the Count method that returns the integer. The Queryable.Count method (as well as the other aggregates and the like) use this method to execute the query ‘right now’.

```C#
public abstract class QueryProvider : IQueryProvider
{
   // ...
   T IQueryProvider.Execute<T>(Expression expression)    // <-----------------------c7
   {                                                   
      return (T)this.Execute(expression);
   }

   object IQueryProvider.Execute(Expression expression)  // return object is ObjectReader<T>
   {
      return this.Execute(expression);
   }
}

public class DbQueryProvider : QueryProvider
{
   private readonly DbConnection connection;
   // ...
   public override object Execute(Expression expression)  // return object is ObjectReader<T>
   {
      DbCommand cmd = this.connection.CreateCommand();
      cmd.CommandText = this.Translate(expression);       
      Type elementType = TypeSystem.GetElementType(expression.Type);

      return Activator.CreateInstance(
          typeof(ObjectReader<>).MakeGenericType(elementType),
          BindingFlags.Instance | BindingFlags.NonPublic, null,
          new object[] { reader },
          null);
   }

   private string Translate(Expression expression)
   {
      return new QueryTranslator().Translate(expression);   
                                                           
   }
}

public class Query<T> : IQueryable<T>
{
   private QueryProvider provider;
   Expression expression;
   
   public IEnumerator<T> GetEnumerator() => ((IEnumerable<T>)this.provider.Execute<AdvanceObjectReader<T>>(this.expression)).GetEnumerator();   // <--------C4
   IEnumerator IEnumerable.GetEnumerator() => ((IEnumerable)this.provider.Execute(this.expression)).GetEnumerator();
   // ...
}
```

## A LINQ to SQL Provider

```C#
//-------------------V
public class Query<T> : IQueryable<T>
{
   private QueryProvider provider;
   Expression expression;

   public Query(QueryProvider provider)
   {    
      this.provider = provider;
      this.expression = Expression.Constant(this);
   }

   public Query(QueryProvider provider, Expression expression)
   {    
      this.provider = provider;
      this.expression = expression;
   }

   Expression IQueryable.Expression => this.expression;
  
   Type IQueryable.ElementType => typeof(T);
  
   IQueryProvider IQueryable.Provider => this.provider;
  
   public IEnumerator<T> GetEnumerator() => ((IEnumerable<T>)this.provider.Execute(this.expression)).GetEnumerator();

   IEnumerator IEnumerable.GetEnumerator() => ((IEnumerable)this.provider.Execute(this.expression)).GetEnumerator();
  
   public override string ToString() => this.provider.GetQueryText(this.expression);
}
//-------------------Ʌ

//---------------------------------V
public abstract class QueryProvider : IQueryProvider
{
   protected QueryProvider() { }

   IQueryable<T> IQueryProvider.CreateQuery<T>(Expression expression)   // this is the method that called by Linq.Queryable.Where()
   {                                                                    // expression here is MethodCallExpression that has two args
      return new Query<T>(this, expression);
   }

   IQueryable IQueryProvider.CreateQuery(Expression expression)
   {
      Type elementType = TypeSystem.GetElementType(expression.Type);

      try
      {
         return (IQueryable)Activator.CreateInstance(typeof(Query<>).MakeGenericType(elementType), new object[] { this, expression });
      }
      catch (TargetInvocationException tie)
      {
         throw tie.InnerException;
      }
   }

   // this is for aggregation fucntion pupose such as Sum(), and we want the value immediately, so we don't enumerate the source. This example mainly implements and uses the non /// generic version, becuase we only demo the usage of Where(). You will see another provider code example in the last section which implements and use this method
   T IQueryProvider.Execute<T>(Expression expression)                                                        
   {                                                     
      throw new NotImplementedException();
   }

   object IQueryProvider.Execute(Expression expression)
   {
      return this.Execute(expression);
   }

   public abstract string GetQueryText(Expression expression);

   public abstract object Execute(Expression expression);
}
//---------------------------------Ʌ

//--------------------------V
public class DbQueryProvider : QueryProvider   // we can't use generic in provider like QueryProvider<T>, because a provider should return different types, see C5
{
   private readonly DbConnection connection;

   public DbQueryProvider(DbConnection connection)
   {
      this.connection = connection;
   }

   public override string GetQueryText(Expression expression)
   {
      return this.Translate(expression);
   }

   public override object Execute(Expression expression)   // the return object could be ObjectReader<Customer>, ObjectReader<Order>  <----------------C5
   {
      DbCommand cmd = this.connection.CreateCommand();
      cmd.CommandText = this.Translate(expression);
      DbDataReader reader = cmd.ExecuteReader();
      Type elementType = TypeSystem.GetElementType(expression.Type);

      return Activator.CreateInstance(
          typeof(ObjectReader<>).MakeGenericType(elementType),
          BindingFlags.Instance | BindingFlags.NonPublic, null,
          new object[] { reader },
          null);
   }

   private string Translate(Expression expression)
   {
      return new QueryTranslator().Translate(expression);
   }
}
//--------------------------Ʌ

//----------------------------V
internal class QueryTranslator : ExpressionVisitor
{
    StringBuilder sb;

    internal string Translate(Expression expression)
    {
        this.sb = new StringBuilder();
        this.Visit(expression);
        return this.sb.ToString();
    }

    private static Expression StripQuotes(Expression e)
    {
        while (e.NodeType == ExpressionType.Quote)
        {
            e = ((UnaryExpression)e).Operand;
        }

        return e;
    }

    protected override Expression VisitMethodCall(MethodCallExpression m)
    {
        if (m.Method.DeclaringType == typeof(Queryable) && m.Method.Name == "Where")   // that's why we pass CachedReflectionInfo.Where_TSource_2(typeof(TSource)) in 2.1
        {
            sb.Append("SELECT * FROM (");
            this.Visit(m.Arguments[0]);
            sb.Append(") AS T WHERE ");
            LambdaExpression lambda = (LambdaExpression)StripQuotes(m.Arguments[1]);
            this.Visit(lambda.Body);
            return m;
        }

        throw new NotSupportedException(string.Format("The method '{0}' is not supported", m.Method.Name));
    }

    protected override Expression VisitUnary(UnaryExpression u)
    {
        switch (u.NodeType)
        {
            case ExpressionType.Not:
                sb.Append(" NOT ");
                this.Visit(u.Operand);
                break;

            default:
                throw new NotSupportedException(string.Format("The unary operator '{0}' is not supported", u.NodeType));
        }

        return u;
    }

    protected override Expression VisitBinary(BinaryExpression b)
    {
        sb.Append("(");

        this.Visit(b.Left);

        switch (b.NodeType)
        {
            case ExpressionType.And:
                sb.Append(" AND ");
                break;

            case ExpressionType.Or:
                sb.Append(" OR");
                break;

            case ExpressionType.Equal:
                sb.Append(" = ");
                break;

            case ExpressionType.NotEqual:
                sb.Append(" <> ");
                break;

            case ExpressionType.LessThan:
                sb.Append(" < ");
                break;

            case ExpressionType.LessThanOrEqual:
                sb.Append(" <= ");
                break;

            case ExpressionType.GreaterThan:
                sb.Append(" > ");
                break;

            case ExpressionType.GreaterThanOrEqual:
               sb.Append(" >= ");
                break;

            default:
                throw new NotSupportedException(string.Format("The binary operator '{0}' is not supported", b.NodeType));
        }

        this.Visit(b.Right);

        sb.Append(")");

        return b;
    }

    protected override Expression VisitConstant(ConstantExpression c)
    {
        IQueryable q = c.Value as IQueryable;   // <-------------------- this is important, C1

        if (q != null)
        {
            // assume constant nodes w/ IQueryables are table references
            sb.Append("SELECT * FROM ");
            sb.Append(q.ElementType.Name);
        }
        else if (c.Value == null)
        {
            sb.Append("NULL");
        }
        else
        {
            switch (Type.GetTypeCode(c.Value.GetType()))
            {
                case TypeCode.Boolean:
                    sb.Append(((bool)c.Value) ? 1 : 0);
                    break;

                case TypeCode.String:
                    sb.Append("'");
                    sb.Append(c.Value);
                    sb.Append("'");
                    break;

                case TypeCode.Object:
                    throw new NotSupportedException(string.Format("The constant for '{0}' is not supported", c.Value));

                default:
                    sb.Append(c.Value);
                    break;
            }
        }

        return c;
    }

    protected override Expression VisitMemberAccess(MemberExpression m)
    {
        if (m.Expression != null && m.Expression.NodeType == ExpressionType.Parameter)
        {
            sb.Append(m.Member.Name);
            return m;
        }

        throw new NotSupportedException(string.Format("The member '{0}' is not supported", m.Member.Name));
    }
}
//----------------------------Ʌ

//-------------------V
internal static class TypeSystem
{
   internal static Type GetElementType(Type seqType)
   {
      Type ienum = FindIEnumerable(seqType);
      if (ienum == null)
         return seqType;

      return ienum.GetGenericArguments()[0];
   }

   private static Type FindIEnumerable(Type seqType)
   {
      if (seqType == null || seqType == typeof(string))
         return null;

      if (seqType.IsArray)
         return typeof(IEnumerable<>).MakeGenericType(seqType.GetElementType());

      if (seqType.IsGenericType)
      {
         foreach (Type arg in seqType.GetGenericArguments())
         {
            Type ienum = typeof(IEnumerable<>).MakeGenericType(arg);
            if (ienum.IsAssignableFrom(seqType))
               return ienum;
         }
      }

      Type[] ifaces = seqType.GetInterfaces();

      if (ifaces != null && ifaces.Length > 0)
      {
         foreach (Type iface in ifaces)
         {
            Type ienum = FindIEnumerable(iface);

            if (ienum != null)
               return ienum;
         }
      }

      if (seqType.BaseType != null && seqType.BaseType != typeof(object))
      {
         return FindIEnumerable(seqType.BaseType);
      }

      return null;
   }
}
//-------------------Ʌ

//----------------------------V
internal class ObjectReader<T> : IEnumerable<T> where T : class, new()
{
   Enumerator enumerator;

   internal ObjectReader(DbDataReader reader)
   {
      this.enumerator = new Enumerator(reader);
   }

   public IEnumerator<T> GetEnumerator()
   {
      Enumerator e = this.enumerator;

      if (e == null)
      {
         throw new InvalidOperationException("Cannot enumerate more than once");
      }

      this.enumerator = null;
      return e;
   }

   IEnumerator IEnumerable.GetEnumerator()
   {
      return this.GetEnumerator();
   }

   class Enumerator : IEnumerator<T>, IDisposable
   {
      DbDataReader reader;
      FieldInfo[] fields;
      int[] fieldLookup;
      T current;

      internal Enumerator(DbDataReader reader)
      {
         this.reader = reader;
         this.fields = typeof(T).GetFields();
      }

      public T Current
      {
         get { return this.current; }
      }

      object IEnumerator.Current
      {
         get { return this.current; }
      }

      public bool MoveNext()
      {
         if (this.reader.Read())
         {
            if (this.fieldLookup == null)
            {
               this.InitFieldLookup();
            }

            T instance = new T();

            for (int i = 0, n = this.fields.Length; i < n; i++)
            {
               int index = this.fieldLookup[i];

               if (index >= 0)
               {
                  FieldInfo fi = this.fields[i];

                  if (this.reader.IsDBNull(index))
                  {
                     fi.SetValue(instance, null);
                  }
                  else
                  {
                     fi.SetValue(instance, this.reader.GetValue(index));
                  }
               }
            }

            this.current = instance;

            return true;
         }

         return false;
      }

      public void Reset()
      {
      }

      public void Dispose()
      {
         this.reader.Dispose();
      }

      private void InitFieldLookup()
      {
         var map = new Dictionary<string, int>(StringComparer.InvariantCultureIgnoreCase);

         for (int i = 0, n = this.reader.FieldCount; i < n; i++)
         {
            map.Add(this.reader.GetName(i), i);
         }

         this.fieldLookup = new int[this.fields.Length];

         for (int i = 0, n = this.fields.Length; i < n; i++)
         {
            int index;

            if (map.TryGetValue(this.fields[i].Name, out index))
            {
               this.fieldLookup[i] = index;
            }
            else
            {
               this.fieldLookup[i] = -1;
            }
         }
      }
   }
}
//----------------------------Ʌ

public class Customer
{
   public string CustomerID;
   public string ContactName;
   public string Phone;
   public string City;
   public string Country;
}

public class Order
{
   public int OrderID;
   public string CustomerID;
   public DateTime OrderDate;
}

public class Northwind
{
   public Query<Customer> Customers;
   public Query<Order> Orders;

   public Northwind(DbConnection connection)
   {
      QueryProvider provider = new DbQueryProvider(connection);   // <---------------------1.1
      this.Customers = new Query<Customer>(provider);            // <---------------------1.2a
      this.Orders = new Query<Order>(provider);
   }
}
```
```C#
static void Main(string[] args)
{
   string constr = @"Server=DESKTOP-KLM1TNG; Database=Northwind; Trusted_Connection=True; MultipleActiveResultSets=true";
   using (SqlConnection con = new SqlConnection(constr))
   {
      con.Open();
      Northwind db = new Northwind(con);   // <---------------------1.0

      IQueryable<Customer> query =
           db.Customers.Where(c => c.City == "London");  // <---------------------2.0
                                                         // every Where() creates a new Query<T> instance, and futher new Query<T> instance contains the first Query<T> instance
                                                         // check 1.2b: this.expression = Expression.Constant(this); this is important, also check C1

       // now `query` is Query<Customer> instance, whose `expression` property is a MethodCallExpression that contains the original Query<Customer> instance as 
       // arg0 (repesented as ConstantExpression), and c => c.City == "London" expression as arg1

      Console.WriteLine(query.Expression.ToString());   // SELECT * FROM Customers.Where(c => (c.City == value(ConsoleAppLinqToSQL.Program+<>c__DisplayClass4_0).myCity))

      foreach (var item in list)    // <------------------------4.0, calls query's GetEnumerator()
      {
         Console.WriteLine("Name: {0}", item.ContactName);
      }

      Console.ReadLine();
   }
}

public static class Queryable 
{
   // ...
   public static IQueryable<TSource> Where<TSource>(this IQueryable<TSource> source, Expression<Func<TSource, bool>> predicate)  // <---------------------2.0
   {
      return source.Provider.CreateQuery<TSource>(
         Expression.Call(    // <---------------------2.1               
            null,
            CachedReflectionInfo.Where_TSource_2(typeof(TSource)),   // return a MethodInfo instance that references Queryable.Where 
            source.Expression,
            Expression.Quote(predicate)
         )
      );
   }
}

public class Query<T> : IQueryable<T>
{
   private QueryProvider provider;
   Expression expression;

   public Query(QueryProvider provider)  // <---------------------1.2b
   {
      this.provider = provider;
      this.expression = Expression.Constant(this);  // <---------- wrap itself for the first time
   }

   public Query(QueryProvider provider, Expression expression)  // <---------------------3.1b
   {                                                            
      this.provider = provider;
      this.expression = expression;
   }

   public IEnumerator<T> GetEnumerator()
   {
      return ((IEnumerable<T>)this.provider.Execute(this.expression)).GetEnumerator();  // <------------------------4.0
   }

   // ...
}

public abstract class QueryProvider : IQueryProvider
{

   IQueryable<T> IQueryProvider.CreateQuery<T>(Expression expression)   // <---------------------3.0, this is the method that called by Linq.Queryable.Where()
   {
      return new Query<T>(this, expression);   // <---------------------3.1a, expression here represents: c => c.City == "London"
   }


   T IQueryProvider.Execute<T>(Expression expression)
   {
      return (T)this.Execute(expression);  
   }

   public abstract string GetQueryText(Expression expression);

   public abstract object Execute(Expression expression);

   // ...
}

public class DbQueryProvider : QueryProvider
{
   private readonly DbConnection connection;

   public DbQueryProvider(DbConnection connection)
   {
      this.connection = connection;
   }

   public override object Execute(Expression expression)   // <------------------------4.1
   {
      DbCommand cmd = this.connection.CreateCommand();
      cmd.CommandText = this.Translate(expression);       // <------------------------5.0a
      DbDataReader reader = cmd.ExecuteReader();          // reader has all the matching records available after the SQL statement is executed
      Type elementType = TypeSystem.GetElementType(expression.Type);

      return Activator.CreateInstance(
          typeof(ObjectReader<>).MakeGenericType(elementType),
          BindingFlags.Instance | BindingFlags.NonPublic, null,
          new object[] { reader },
          null);
   }

   private string Translate(Expression expression)
   {
      return new QueryTranslator().Translate(expression);   // <------------------------5.0b
                                                            // QueryTranslator is a ExpressionVisitor that Visit all the nodes to genenrate SQL statment string
   }
}
```
 
One of the problem we have for the above provider is, we can't use local variable:

```C#
string city = "London";

var query = db.Customers.Where(c => c.City == city);

Console.WriteLine(query.Expression.ToString());  // throws an exception: The member 'city' is not supported
```

it is because of the code we have in the `QueryTranslator`:

```C#
protected override Expression VisitMemberAccess(MemberExpression m)
{
   if (m.Expression != null && m.Expression.NodeType == ExpressionType.Parameter)
   {
      sb.Append(m.Member.Name);
      return m;
   }

   throw new NotSupportedException(string.Format("The member '{0}' is not supported", m.Member.Name));  
   // we deliberately throw exception when MemberExpression's NodeType is not ExpressionType.Parameter,
   // because we want to take care of the sepcial MemberExpression node with a ConstantExpression in its Expression property
   // why we want to take specail care of it? Remember compiler generate a on the fly class and create an instance of that to represent MemberExpression
   // check "How Compiler Handler Local Variable" for details, how is our translator going to access this value?
}
```

You should be clear about the fix, we can do sth like:

```C#
protected override Expression VisitMemberAccess(MemberExpression m)
{
   if (m.Expression != null && m.Expression.NodeType == ExpressionType.Parameter)
   {
      sb.Append(m.Member.Name);
      return m;
   }
   
   if (m.Expression.NodeType == ExpressionType.Constant) 
   {
      // we can wrap MemberExpression m into LambdaExpression then compile it then Invoke the delegate to get the constant value, that's exactly what we are going to do
      // in Evaluator covered in the next. The reason we don't do it here because it is not a generic fix, and it is not good to compile LambdaExpression in Visitor
   }
}
```

Below is the solution:

don't worry about the details, it involves some logic like cache the Expressions that need to be evaluated and have some logic like if child node cannot be evaluated, then 
parent node should not be evaluated too etc, we just focus on the core part of this solution, which is to wrap eligible `MemberExpression` and then complie it to get the represented `Delegate`, then invoke the delegate to get the value then wrap the value into a `ConstantExpression`

```C#
public static class Evaluator
{
   public static Expression PartialEval(Expression expression, Func<Expression, bool> fnCanBeEvaluated)
   {
      return new SubtreeEvaluator(new Nominator(fnCanBeEvaluated).Nominate(expression))
                 .Eval(expression);
   }

   public static Expression PartialEval(Expression expression)
   {
      return PartialEval(expression, Evaluator.CanBeEvaluatedLocally);
   }

   private static bool CanBeEvaluatedLocally(Expression expression)
   {
      return expression.NodeType != ExpressionType.Parameter;
   }
}

public class SubtreeEvaluator : ExpressionVisitor
{
   HashSet<Expression> candidates;

   internal SubtreeEvaluator(HashSet<Expression> candidates)
   {
      this.candidates = candidates;
   }

   internal Expression Eval(Expression exp)
   {
      return this.Visit(exp);
   }

   public override Expression Visit(Expression exp)
   {
      if (exp == null)
         return null;

      if (this.candidates.Contains(exp))
         return this.Evaluate(exp);

      return base.Visit(exp);
   }

   private Expression Evaluate(Expression e)
   {
      if (e.NodeType == ExpressionType.Constant)
         return e;

      // 
      LambdaExpression lambda = Expression.Lambda(e);   // <------------------------------ the most essential part of this solution
      Delegate fn = lambda.Compile();
      //
      return Expression.Constant(fn.DynamicInvoke(null), e.Type);
   }
}

public class Nominator : ExpressionVisitor
{
   Func<Expression, bool> fnCanBeEvaluated;
   HashSet<Expression> candidates;
   bool cannotBeEvaluated;

   internal Nominator(Func<Expression, bool> fnCanBeEvaluated)
   {
      this.fnCanBeEvaluated = fnCanBeEvaluated;
   }

   internal HashSet<Expression> Nominate(Expression expression)
   {
      this.candidates = new HashSet<Expression>();
      this.Visit(expression);
      return this.candidates;
   }

   public override Expression Visit(Expression expression)
   {
      if (expression != null)
      {
         bool saveCannotBeEvaluated = this.cannotBeEvaluated;
         this.cannotBeEvaluated = false;
         base.Visit(expression);

         if (!this.cannotBeEvaluated)
         {
            if (this.fnCanBeEvaluated(expression))
            {
               this.candidates.Add(expression);
            }
            else
            {
               this.cannotBeEvaluated = true;
            }
         }

         this.cannotBeEvaluated |= saveCannotBeEvaluated;
      }

      return expression;
   }
}
```

Now you can use it in the provider:

```C#
public class DbQueryProvider : QueryProvider
{
   // ...
   private string Translate(Expression expression) 
   {
      expression = Evaluator.PartialEval(expression);  // after execute it, the expression's all MemberExpression nodes becomes ConstantExpression node
                                                       // which can be analysed by the translator easily
      return new QueryTranslator().Translate(expression);
   }
}
```

Another provider that supports aggregation fucntion:

https://weblogs.asp.net/dixin/understanding-linq-to-sql-10-implementing-linq-to-sql-provider
```C#
public class QueryProvider : IQueryProvider
{
    // Translates LINQ query to SQL.
    private readonly Func<IQueryable, DbCommand> _translator;

    // Executes the translated SQL and retrieves results.
    private readonly Func<Type, string, object[], IEnumerable> _executor;

    public QueryProvider(
        Func<IQueryable, DbCommand> translator,
        Func<Type, string, object[], IEnumerable> executor)
    {
        this._translator = translator;
        this._executor = executor;
    }

    public IQueryable<TElement> CreateQuery<TElement>(Expression expression)
    {
        return new Queryable<TElement>(this, expression);
    }

    public IQueryable CreateQuery(Expression expression)
    {
        throw new NotImplementedException();
    }

    public TResult Execute<TResult>(Expression expression)   // support both aggregration and non aggregration functions
    {
        bool isCollection = typeof(TResult).IsGenericType &&
            typeof(TResult).GetGenericTypeDefinition() == typeof(IEnumerable<>);  // <-----------------   
        // ...
        return isCollection ? (TResult)queryResult : queryResult.OfType<TResult>().SingleOrDefault(); // Returns a single item.
    }

    public object Execute(Expression expression)  // <----------------unlike the previous provider example, this provider only uses the generic one
    {
        throw new NotImplementedException();
    }
}
```


<style type="text/css">
.markdown-body {
  max-width: 1800px;
  margin-left: auto;
  margin-right: auto;
}
</style>

<link rel="stylesheet" href="./zCSS/bootstrap.min.css">
<script src="./zCSS/jquery-3.3.1.slim.min.js"></script>
<script src="./zCSS/popper.min.js"></script>
<script src="./zCSS/bootstrap.min.js"></script>
