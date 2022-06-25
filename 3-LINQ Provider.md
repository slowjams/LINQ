## LINQ Provider

===============================================

A LINQ to SQL Provider:

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

   T IQueryProvider.Execute<T>(Expression expression) 
   {
      return (T)this.Execute(expression);
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
public class DbQueryProvider : QueryProvider
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

   public override object Execute(Expression expression)
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

public class Customers
{
   public string CustomerID;
   public string ContactName;
   public string Phone;
   public string City;
   public string Country;
}

public class Orders
{
   public int OrderID;
   public string CustomerID;
   public DateTime OrderDate;
}

public class Northwind
{
   public Query<Customers> Customers;
   public Query<Orders> Orders;

   public Northwind(DbConnection connection)
   {
      QueryProvider provider = new DbQueryProvider(connection);   // <---------------------1.1
      this.Customers = new Query<Customers>(provider);            // <---------------------1.2a
      this.Orders = new Query<Orders>(provider);
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

      IQueryable<Customers> query =
           db.Customers.Where(c => c.City == "London");  // <---------------------2.0
                                                         // every Where() creates a new Query<T> instance, and futher new Query<T> instance contains the first Query<T> instance
                                                         // check 1.2b: this.expression = Expression.Constant(this); this is important, also check C1

       // now `query` is Query<Customers> instance, whose `expression` property is a MethodCallExpression that contains the original Query<Customers> instance as 
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

it is because of the code we have in:

```C#
protected override Expression VisitMember(MemberExpression m)
{
   if (m.Expression != null && m.Expression.NodeType == ExpressionType.Parameter)
   {
      sb.Append(m.Member.Name);
      return m;
   }

   throw new NotSupportedException(string.Format("The member '{0}' is not supported", m.Member.Name));  // <-----------
}
```

-------------------------------------------------------------------------------------------------------------------
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

      LambdaExpression lambda = Expression.Lambda(e);

      Delegate fn = lambda.Compile();

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

object? Execute(Expression expression);  // The <see cref="Execute"/> method executes queries that return a single value (instead of an enumerable sequence of values). Expression trees that represent queries that return enumerable results are executed when their associated <see cref="IQueryable"/> object is enumerated.    

Figure it out what happen when you call Where on IEnumerable<T> more than once



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
