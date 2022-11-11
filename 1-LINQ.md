## LINQ

=====================================

Source Code:

```C#
//------------------------------------V
public static partial class Enumerable 
{
   public static IEnumerable<TSource> Where<TSource>(this IEnumerable<TSource> source, Func<TSource, bool> predicate);

   public static IEnumerable<TSource> Where<TSource>(this IEnumerable<TSource> source, Func<TSource, int, bool> predicate);  // using index is not supported by query expression

   public static IEnumerable<TResult> Select<TSource, TResult>(this IEnumerable<TSource> source, Func<TSource, TResult> selector) {
      if (source is Iterator<TSource> iterator) 
      {
         return iterator.Select(selector);
      }

      if (source is IList<TSource> ilist)
      {
         if (source is TSource[] array)
         {
            return array.Length == 0 ? Empty<TResult>() : new SelectArrayIterator<TSource, TResult>(array, selector);
         }

         if (source is List<TSource> list)
         {
            return new SelectListIterator<TSource, TResult>(list, selector);
         }

         return new SelectIListIterator<TSource, TResult>(ilist, selector);
      }

      return new SelectEnumerableIterator<TSource, TResult>(source, selector);
   }

   public static IEnumerable<TResult> SelectMany<TSource, TResult>(this IEnumerable<TSource> source, Func<TSource, IEnumerable<TResult>> selector);

   public static IEnumerable<TResult> SelectMany<TSource, TCollection, TResult>(this IEnumerable<TSource> source, 
                                                                                Func<TSource, IEnumerable<TCollection>> collectionSelector, 
                                                                                Func<TSource, TCollection, TResult> resultSelector);
   
   public static IEnumerable<TResult> Join<TOuter, TInner, TKey, TResult>(this IEnumerable<TOuter> outer, 
                                                                          IEnumerable<TInner> inner, 
                                                                          Func<TOuter, TKey> outerKeySelector,  
                                                                          Func<TInner, TKey> innerKeySelector, 
                                                                          Func<TOuter, TInner, TResult> resultSelector);

   //--------------------------------V    
   public static IEnumerable<IGrouping<TKey, TSource>> GroupBy<TSource, TKey>(this IEnumerable<TSource> source, Func<TSource, TKey> keySelector); 

   public static ILookup<TKey, TElement> ToLookup<TSource, TKey, TElement>(this IEnumerable<TSource> source, Func<TSource, TKey> keySelector);  // more advanced than GroupBy
  
   public interface ILookup<TKey, TElement> : IEnumerable<IGrouping<TKey, TElement>> 
   {
      int Count { get; }
      IEnumerable<TElement> this[TKey key] { get; }
      bool Contains(TKey key);
   }
   //--------------------------------Ʌ
   public static IEnumerable<TResult> GroupJoin<TOuter, TInner, TKey, TResult>(this IEnumerable<TOuter> outer, 
                                                                               IEnumerable<TInner> inner, 
                                                                               Func<TOuter, TKey> outerKeySelector,
                                                                               Func<TInner, TKey> innerKeySelector, 
                                                                               Func<TOuter, IEnumerable<TInner>, TResult> resultSelector);

   public static IOrderedEnumerable<TSource> OrderBy<TSource, TKey>(this IEnumerable<TSource> source, Func<TSource, TKey> keySelector);
   
   public static IOrderedEnumerable<TSource> OrderByDescending<TSource, TKey>(this IEnumerable<TSource> source, Func<TSource, TKey> keySelector); 
            
   public static IOrderedEnumerable<TSource> ThenBy<TSource, TKey>(this IOrderedEnumerable<TSource> source, Func<TSource, TKey> keySelector, IComparer<TKey>? comparer);   

   public static IOrderedEnumerable<TSource> ThenByDescending<TSource, TKey>(this IOrderedEnumerable<TSource> source, Func<TSource, TKey> keySelector, IComparer<TKey>? comparer);
                                                                  
   // ...
   public static IEnumerable<TSource> Take<TSource>(this IEnumerable<TSource> source, int count)
   {
      return count <= 0 ? Empty<TSource>() : TakeIterator<TSource>(source, count);   
   }

   public static bool All<TSource>(this IEnumerable<TSource> source, Func<TSource, bool> predicate) // Any works unexpected, it return true when source is null when source is empty
   {
      foreach (TSource element in source) {   // <------------------- if source is empty, it doesn't enumerate in foreach, so it return true, which we need to pay attention to 
         if (!predicate(element)) {
            return false;
         }
      }

      return true;
   }

   public static bool Any<TSource>(this IEnumerable<TSource> source, Func<TSource, bool> predicate)   // Any works expected when the source is empty
   {
      foreach (TSource element in source) {
         if (predicate(element)) {
            return true;
         }
      }

      return false;
   }

   public static IEnumerable<TSource> Distinct<TSource>(this IEnumerable<TSource> source, IEqualityComparer<TSource> comparer);  // check CLR via C# Walkthrough Chapter 5
 
   //--------------------------------V
   public static IEnumerable<TResult> Empty<TResult>()
   {
      return EmptyEnumerable<TResult>.Instance;
   }

   internal class EmptyEnumerable<TElement>
   {
      public static readonly TElement[] Instance = new TElement[0];
   }
   //--------------------------------Ʌ

   //--------------------------------V
   public IEnumerable<T> OfType<T>(this IEnumerable source)   // doesn't throw exception
   {
      foreach(object o in source) 
      {
         if(o is T) 
         {
            yield return (T) o;
         }     
      }
      
   }

   public IEnumerable<T> Cast<T>(this IEnumerable source) {  // throw exception when the curr item cannot be casted 
      foreach(object o in source) 
      {
         yield return (T) o;
      }        
   }
   //--------------------------------Ʌ

   public static TSource Aggregate<TSource>(this IEnumerable<TSource> source, Func<TSource, TSource, TSource> func)  // use first item in the list as starting point
   {
      using (IEnumerator<TSource> e = source.GetEnumerator()) 
      {
         if (!e.MoveNext())
            ThrowHelper.ThrowNoElementsException();

         TSource result = e.Current;
         while (e.MoveNext()) 
         {
            result = func(result, e.Current);   // e.Current is actually "next"
         }

         return result;  
      }
   }

   public static TAccumulate Aggregate<TSource, TAccumulate>(this IEnumerable<TSource> source, TAccumulate seed, Func<TAccumulate, TSource, TAccumulate> func)
   {
      TAccumulate result = seed;

      foreach (TSource element in source)
      {
         result = func(result, element);
      }

      return result;
   }

   public static TResult Aggregate<TSource, TAccumulate, TResult>(this IEnumerable<TSource> source, 
                                                                  TAccumulate seed, 
                                                                  Func<TAccumulate, TSource, TAccumulate> func, 
                                                                  Func<TAccumulate, TResult> resultSelector)     // doesn' look very useful
   {
      TAccumulate result = seed;
      
      foreach (TSource element in source) 
      {
         result = func(result, element);
      }

      return resultSelector(result);
   }
}
//------------------------------------Ʌ

//---------------------------------------VV
internal abstract class Iterator<TSource> : IEnumerable<TSource>, IEnumerator<TSource> 
{
   private readonly int _threadId;
   internal int _state;
   internal TSource _current = default!;

   protected Iterator() {
      _threadId = Environment.CurrentManagedThreadId;
   }

   public TSource Current => _current;

   public abstract Iterator<TSource> Clone();  // for scenarios that you need to enumerate the collection multiple times concurrently

   public virtual void Dispose()
   {
      _current = default!;
      _state = -1;
   }

   public IEnumerator<TSource> GetEnumerator() 
   {
      Iterator<TSource> enumerator = _state == 0 && _threadId == Environment.CurrentManagedThreadId ? this : Clone();
      enumerator._state = 1;
      return enumerator;
   }

   public abstract bool MoveNext();

   public virtual IEnumerable<TResult> Select<TResult>(Func<TSource, TResult> selector) 
   {
      return new SelectEnumerableIterator<TSource, TResult>(this, selector);
   }

   public virtual IEnumerable<TSource> Where(Func<TSource, bool> predicate)
   {
      return new WhereEnumerableIterator<TSource>(this, predicate);
   }

   object? IEnumerator.Current => Current;

   IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

   void IEnumerator.Reset() => ThrowHelper.ThrowNotSupportedException();
}
//---------------------------------------ɅɅ

//---------------------------------------------------------------V
private sealed partial class SelectListIterator<TSource, TResult> : Iterator<TResult> 
{
   private readonly List<TSource> _source;
   private readonly Func<TSource, TResult> _selector;
   private List<TSource>.Enumerator _enumerator;

   public SelectListIterator(List<TSource> source, Func<TSource, TResult> selector) {
      _source = source;
      _selector = selector;
   }

   public override Iterator<TResult> Clone() => new SelectListIterator<TSource, TResult>(_source, _selector);  // in case you need to call foreach multiple times

   public override bool MoveNext()
   {
      switch (_state)
      {
         case 1:
            _enumerator = _source.GetEnumerator();
            _state = 2;
            goto case 2;
         case 2:
            if (_enumerator.MoveNext())
            {
               _current = _selector(_enumerator.Current);
               return true;
            }

            Dispose();
            break;
      }

      return false;
   }

   public override IEnumerable<TResult2> Select<TResult2>(Func<TResult, TResult2> selector)
   {
      return new SelectListIterator<TSource, TResult2>(_source, CombineSelectors(_selector, selector));
   }
}
//---------------------------------------------------------------Ʌ
```


## Introduction to LINQ

All query expressions begin with a `from` clause and end with a `select` or `group-by` caluse


## From Clause

`From` defines a data source that can be `IEnumerable<T>` or `IQueryable<T>` (`(which implements IEnumerable<T>`):

```C#
List<Customer> customers = new List<Customer> { 
    new Customer { Name = "Paolo", City = "Brescia",
        Orders = new Order[] {
            new Order { IdOrder = 1, EuroAmount = 100, Description = "Order 1" },
            new Order { IdOrder = 2, EuroAmount = 150, Description = "Order 2" },
            new Order { IdOrder = 3, EuroAmount = 230, Description = "Order 3" },
        }},
    new Customer { Name = "Marco", City = "Torino",
        Orders = new Order[] {
           new Order { IdOrder = 4, EuroAmount = 320, Description = "Order 4" },
          new Order { IdOrder = 5, EuroAmount = 170, Description = "Order 5" },
        }
}};

from c in customers

from Customer c in customers  // not recommended, it calls `Cast<T>()` method, which is a performance hit
```

The `from` clause doens't get translated into a method call, but LINQ can have multiple `from` clauses, **so any subsequent `from` are translated into `SelectMany`**:

```C#
var ordersQuery =            // ordersQuery has 5 (pure) Order instances
    from c in customers
    from o in c.Orders       // o is just a placeholder
    select o;

Enumerable<Order> orders = customers.SelectMany(c => c.Orders);

var ordersQuery =            // ordersQuery has 5 anonymous objects, each object contains one cust property which in turn has refernce to all its order 
    from c in customers      // and one order instance. For the cust property, it's like a cross join with its belonging orders
    from o in c.Orders
    select new { c, o };

var a = customers.SelectMany(c => c.Orders, (c, o) => new { c, o } );
```

You can also do nested in `from`:

```C#
 from x in from XXX in XXXX
```

## Group-by Clause

```C#
public static partial class Enumerable 
{
   public static IEnumerable<IGrouping<TKey, TSource>> GroupBy<TSource, TKey>(this IEnumerable<TSource> source, Func<TSource, TKey> keySelector) 
   {
      return new GroupedEnumerable<TSource, TKey>(source, keySelector, null);;
   }     
   // ...
   public static IEnumerable<IGrouping<TKey, TElement>> GroupBy<TSource, TKey, TElement>(this IEnumerable<TSource> source, Func<TSource, TKey> keySelector, 
                                                                                         Func<TSource, TElement> elementSelector, 
                                                                                         IEqualityComparer<TKey>? comparer) 
   {
      return new GroupedEnumerable<TSource, TKey, TElement>(source, keySelector, elementSelector, comparer);
   }
            
   internal sealed partial class GroupedEnumerable<TSource, TKey, TElement> : IEnumerable<IGrouping<TKey, TElement>> {
       private readonly IEnumerable<TSource> _source;
       private readonly Func<TSource, TKey> _keySelector;
       private readonly Func<TSource, TElement> _elementSelector;
       private readonly IEqualityComparer<TKey>? _comparer;
       // ...
       public IEnumerator<IGrouping<TKey, TElement>> GetEnumerator() 
       {
          return Lookup<TKey, TElement>.Create(_source, _keySelector, _elementSelector, _comparer).GetEnumerator();   // complicated, a lot of while loop
       }

       IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();   
   }
}

public partial class Lookup<TKey, TElement> : ILookup<TKey, TElement> 
{
   private readonly IEqualityComparer<TKey> _comparer;
   private Grouping<TKey, TElement>[] _groupings;
   private Grouping<TKey, TElement>? _lastGrouping;
   private int _count;
   // ...
}

public interface IGrouping<TKey,TElement> : IEnumerable<TElement>  // <-----------------------------
{
   TKey Key { get; }
}

public class Grouping<TKey, TElement> : IGrouping<TKey, TElement>, IList<TElement> 
{
   internal readonly TKey _key;
   internal readonly int _hashCode;
   internal TElement[] _elements;   // <-------------
   internal int _count;
   internal Grouping<TKey, TElement>? _hashNext;
   internal Grouping<TKey, TElement>? _next;
   // ...
   internal void Add(TElement element);

   public IEnumerator<TElement> GetEnumerator() {
      for (int i = 0; i < _count; i++) {
         yield return _elements[i];
      }
   }
}
```

Example:

```C#
public interface IGrouping<TKey,TElement> : IEnumerable<TElement>
{
   TKey Key { get; }
}

List<Developer> developers = new List<Developer>() {
   new Developer { Name = "Michael", Language = "C#", Title = "Senior" },
   new Developer { Name = "Matt", Language = "C#", Title = "Mid" },
   new Developer { Name = "John", Language = "C#", Title = "Junior" },
   new Developer { Name = "Tom", Language = "Javascript", Title = "Senior" },
   new Developer { Name = "Andy", Language = "Javascript", Title = "Junior" },
   new Developer { Name = "Justin", Language = "SQL", Title = "Grad" },
};

//---------V
var q = from d in developers
        group d by d.Language;  // key is always after `by` keyword <---------------------------------------------

                                                                 // key selector
IEnumerable<IGrouping<string, Developer>> q = developers.GroupBy(d => d.Language);  // a contains 3 IGrouping instance, each instance has Key property and Developer[],
                                                                                    // since there is no element selector, the default one `Developer` is used as element
/*
[0] { Key = "C#", IGrouping<string, Developer> { Michael, Matt, John }  }
[1] { Key = "Javascript", IGrouping<string, Developer> { Tom, Andy } }
[2] { Key = "SQL", IGrouping<string, Developer> { Justin } }
*/
//---------Ʌ

//---------V
var grouped = from d in developers
              group d.Name by d.Language;  // element is always between `group` and `by` keywords  <--------------------

                                                          // key selector      element selector
IEnumerable<IGrouping<string, string>> b = developers.GroupBy(d => d.Language, d => d.Name);  // b also contains 3 IGrouping instance, and use Name string as element
//---------Ʌ
```

You can also group anonymous type:

```C#
//---------V
var grouped = from d in developers
              group d by new { d.Language, d.Title };

                           // key selector, key is anonymous type
var a = developers.GroupBy(d => new { d.Language, d.Title });  
//---------Ʌ

//---------V
var grouped = from d in developers
              group new { d.Name } by new { d.Language, d.Title };

var a = developers.GroupBy(d => new { d.Language, d.Title }, d => new { d.Name });
//---------Ʌ
```


## OrderBy Clause

```C#
public static partial class Enumerable
{
   public static IOrderedEnumerable<TSource> OrderBy<TSource, TKey>(this IEnumerable<TSource> source, Func<TSource, TKey> keySelector) 
   {
      return new OrderedEnumerable<TSource, TKey>(source, keySelector, null, false, null);
   }

   public static IOrderedEnumerable<TSource> OrderBy<TSource, TKey>(this IEnumerable<TSource> source, Func<TSource, TKey> keySelector, IComparer<TKey>? comparer)
   {
      return new OrderedEnumerable<TSource, TKey>(source, keySelector, comparer, false, null);
   }

   public static IOrderedEnumerable<TSource> OrderByDescending<TSource, TKey>(this IEnumerable<TSource> source, Func<TSource, TKey> keySelector, IComparer<TKey>? comparer)
   {
      return new OrderedEnumerable<TSource, TKey>(source, keySelector, comparer, true, null);
   }
            
   public static IOrderedEnumerable<TSource> ThenBy<TSource, TKey>(this IOrderedEnumerable<TSource> source, Func<TSource, TKey> keySelector, IComparer<TKey>? comparer)
   {
      return source.CreateOrderedEnumerable(keySelector, comparer, false);
   }     

   public static IOrderedEnumerable<TSource> ThenByDescending<TSource, TKey>(this IOrderedEnumerable<TSource> source, Func<TSource, TKey> keySelector, IComparer<TKey>? comparer);

   // ...   
}

public interface IOrderedEnumerable<TElement> : IEnumerable<TElement>
{
   IOrderedEnumerable<TElement> CreateOrderedEnumerable<TKey>(Func<TElement, TKey> keySelector, IComparer<TKey> comparer, bool descending);
}

internal abstract partial class OrderedEnumerable<TElement> : IOrderedEnumerable<TElement> 
{
   internal IEnumerable<TElement> _source;

   protected OrderedEnumerable(IEnumerable<TElement> source) => _source = source;
   // ...
   public IEnumerator<TElement> GetEnumerator();
   // ...
}

internal sealed class OrderedEnumerable<TElement, TKey> : OrderedEnumerable<TElement> {
   private readonly OrderedEnumerable<TElement>? _parent;
   private readonly Func<TElement, TKey> _keySelector;
   private readonly IComparer<TKey> _comparer;
   private readonly bool _descending;
 
   // ...
}
```

Example:

```C#
// Good: only sort once using composition
var a = from d in developers
        orderby d.Age, d.Name descending, d.Language
        select d;

var a = developers.OrderBy(d => d.Age)
                  .ThenByDescending(d => d.Name)
                  .ThenBy(d => d.Language).
                  .Select(d => d);
```

Note that don't use multiple `orderby` even though it will get the same result but the performance is bad:

```C#
// Bad: sort three times, bad performance

var a = from d in developers
        orderby d.Age, 
        orderby d.Name, 
        orderby d.Language
        select d;

var a = developers.OrderBy(d => d.Age)
                  .OrderBy(d => d.Name)
                  .OrderBy(d => d.Language).
                  .Select(d => d);

IOrderedEnumerable<Developer> a = developers.OrderBy(d => d.Age);
IOrderedEnumerable<Developer> b = b.OrderBy(d => d.Age);
```

Note that compiler is smart to "wipe out" `Select` clause if the `OrderBy` and `Select` operate on the same thing:

```C#
var a = from d in developers
        orderby d
        select d;

var b = from d in developers
        orderby d
        select d.ToString();
```

You might think both `a` and `b` are `IEnumerable<int>`, however, a is `IOrderedEnumerable<int>`:

```C#
IOrderedEnumerable<int> a = from d in developers  // the reason why a is IOrderedEnumerable<T> is that, itself implements IEnumerable<T>, so compiler optimized it 
                            orderby d             // by not translating the last select caluse into Select() method
                            select d;

IEnumerable<int> b = from d in developers
                     orderby d
                     select d.ToString();   // ToString change int to string, which is a different type to d (Developer)
```


## Query Continuations: `into`

Both `select` and `group by` can be followed by `into` identifier, which is known as a *query continuation*. There is tranlsation in method call, and you can think of it as using a temporary variable:

```C#
// original query
var query = from x in people                 
            select x.Name into y
            select y.ToUpper();

// you can think query continuation translation into a temp variable as:
var tmp = from x in people
          select x.Name;

var query = from y in tmp         
            select y.ToUpper();

// final translation into methods as:
var query = people.Select(x => x.Name)
                  .Select(y => y.ToUpper());

// you can also think it is nested from by moving them as:
var query = from y in (from x in people select x.Name)
            select y.ToUpper();

// not really need parentheses
var query = from y in from x in people select x.Name
            select y.ToUpper();
```


## Join Clause

`Join` is equivalent to "inner join" in SQL

```C#
public static partial class Enumerable
{
   public static IEnumerable<TResult> Join<TOuter, TInner, TKey, TResult>(this IEnumerable<TOuter> outer, IEnumerable<TInner> inner, Func<TOuter, TKey> outerKeySelector,  
                                                                          Func<TInner, TKey> innerKeySelector, Func<TOuter, TInner, TResult> resultSelector);
   
   public static IEnumerable<TResult> Join<TOuter, TInner, TKey, TResult>(this IEnumerable<TOuter> outer, IEnumerable<TInner> inner, Func<TOuter, TKey> outerKeySelector,  
                                                                          Func<TInner, TKey> innerKeySelector, Func<TOuter, TInner, TResult> resultSelector, IEqualityComparer<TKey>? comparer);
}
```

```C#
public class Category
{
   public Int32 IdCategory { get; set; }
   public String Name { get; set; }
}

public class Product
{
   public String IdProduct { get; set; }
   public Int32 IdCategory { get; set; }
   public String Description { get; set; }
}

Category[] categories = new Category[] {
   new Category { IdCategory = 1, Name = "Pasta"},
   new Category { IdCategory = 2, Name = "Beverages"},
   new Category { IdCategory = 3, Name = "Other food"},
};

Product[] products = new Product[] {
   new Product { IdProduct = "PASTA01", IdCategory = 1, Description = "Tortellini" },
   new Product { IdProduct = "PASTA02", IdCategory = 1, Description = "Spaghetti" },
   new Product { IdProduct = "PASTA03", IdCategory = 1, Description = "Fusilli" },
   new Product { IdProduct = "BEV01", IdCategory = 2, Description = "Water" },
   new Product { IdProduct = "BEV02", IdCategory = 2, Description = "Orange Juice" },
};

/*
join identifier in inner-sequence on outer-key-selector equals inner-key-selector
*/

var categoriesAndProducts = from c in categories                               
                            join p in products on c.IdCategory equals p.IdCategory  //  SQL let you swap operands, but query expression require "outter equals inner"
                            select new {
                               CategoryID = c.IdCategory,
                               CategoryName = c.Name,
                               Product = p.Description
                            };

IEnumerable<a`> a = categories.Join(products, c => c.IdCategory, p => p.IdCategory, (c, p) => new { c.IdCategory, CategoryName = c.Name, Product = p.Description });

/*
[0] { CategoryID = 1, CategoryName = "Pasta", Product = "Tortellini" }
[1] { CategoryID = 1, CategoryName = "Pasta", Product = "Spaghetti" }
[2] { CategoryID = 1, CategoryName = "Pasta", Product = "Fusilli" }
[3] { CategoryID = 2, CategoryName = "Beverages", Product = "Water" }
[4] { CategoryID = 2, CategoryName = "Beverages", Product = "Orange Juice" }
 */
```

let's change the data and make it no result and see it is really "inner join":

```C#
Category[] categories = new Category[] {
   new Category { IdCategory = 11, Name = "Pasta"},
   new Category { IdCategory = 22, Name = "Beverages"},
   new Category { IdCategory = 33, Name = "Other food"},
};
// products stay the same

var q1 = from c in categories
         join p in products on c.IdCategory equals p.IdCategory          
         select p;

var q2 = from c in categories
         join p in products on c.IdCategory equals p.IdCategory          
         select c;

// both q1 and q2 are "Empty", which is like:
var q1 = Enumerable.Empty<Product>();

var q2 = Enumerable.Empty<Category>();
```

`GroupJoin` ("join into") is equivalent to "left outter join" in in SQL: (well, not quite, but 90%, to make it 100%, use `DefaultIfEmpty`)

```C#
//retn typ is IGrouping<TKey, TElement>  <------------------------------------
public static IEnumerable<TResult> GroupJoin<TOuter, TInner, TKey, TResult>(this IEnumerable<TOuter> outer, 
                                                                            IEnumerable<TInner> inner, 
                                                                            Func<TOuter, TKey> outerKeySelector,
                                                                            Func<TInner, TKey> innerKeySelector, 
                                                                            Func<TOuter, IEnumerable<TInner>, TResult> resultSelector)
{
   using (IEnumerator<TOuter> e = outer.GetEnumerator()) {
      if (e.MoveNext()) {
         Lookup<TKey, TInner> lookup = Lookup<TKey, TInner>.CreateForJoin(inner, innerKeySelector, comparer);
         do {
            TOuter item = e.Current;
            yield return resultSelector(item, lookup[outerKeySelector(item)]);   //  lookup's index return an instance of `Grouping<TKey, TElement>`
         }
         while (e.MoveNext());
      }
    }
}
                                                                  
```
```C#
//---------------V
var q = from c in categories                                     // IEnumerable<Product>  
        join p in products on c.IdCategory equals p.IdCategory into productsByCategory     //productsByCategory is actually `IGrouping<string, Product>`
        select productsByCategory;

IEnumerable<IEnumerable<Product>> q = categories.GroupJoin(products, c => c.IdCategory, p => p.IdCategory, (c, productsByCategory) => productsByCategory);

// it is the same as below
IEnumerable<IGrouping<String, Product>> q = ...;  // even though IGrouping does inherit IEnumerable<out T>, but you can't do casting here becuase of the covariant `out` keyword

/* q contains 3 instances

[0] IGrouping<string, Product> instance { Key = 1, Contains 3 product: Tortellini, Spaghetti, Fusilli }
[1] IGrouping<string, Product> instance { Key = 2, Contains 2 product: Water, Orange Juice}
[2] IGrouping<string, Product> instance { yeild no result}


note that [2] is actually EmptyPartition<Product> instance, I just use IGrouping to to simplify
 */
//---------------Ʌ

//---------------V
var q = from c in categories.
        join p in products on c.IdCategory equals p.IdCategory into productsByCategory
        from pc in productsByCategory    // second from get translated into SelectMany
        select pc;

// compiler translates the query into:
var q = categories.GroupJoin(products, c => c.IdCategory, p => p.IdCategory,  (c, productsByCategory) => new { c, productsByCategory}) // make you can use c in query expression
                  .SelectMany(anon => anon.productsByCategory, (anon, pc) => pc);    // anon just means anonymous type

// but it is equivalent to this one below, because we don't really need category information 
IEnumerable<Product> q = categories.GroupJoin(products, c => c.IdCategory, p => p.IdCategory, (c, productsByCategory) => productsByCategory).SelectMany(g => g);
//------------------Ʌ

//----------------V
 var q = from c in categories                                     
         join p in products on c.IdCategory equals p.IdCategory into productsByCategory    
         select new {                                            
            CategoryID = c.IdCategory,
            CategoryName = c.Name,          // you can access c here because of the projection 
            Products = productsByCategory
         };
                                                                              // productsByCategory is IEnumerable<Product> 
var q = categories.GroupJoin(products, c => c.IdCategory, p => p.IdCategory, (c, productsByCategory) => new { c.IdCategory, CategoryName = c.Name, Products = productsByCategory });

/*
[0] { CategoryID = 1, CategoryName = "Pasta", Products = {                 Grouping<string, Product> that contains following instances
                                                 new Product { IdProduct = "PASTA01", IdCategory = 1, Description = "Tortellini" },
                                                 new Product { IdProduct = "PASTA02", IdCategory = 1, Description = "Spaghetti" },
                                                 new Product { IdProduct = "PASTA03", IdCategory = 1, Description = "Fusilli" }
                                              }}

[1] { CategoryID = 2, CategoryName = "Beverages", Products = {             Grouping<string, Product>
                                                    new Product { IdProduct = "BEV01", IdCategory = 2, Description = "Water" },
                                                    new Product { IdProduct = "BEV02", IdCategory = 2, Description = "Orange Juice" }
                                                  }}

[2] { CategoryID = 3, CategoryName = "Other food", Products = {            EmptyPartition<Product>, this is the "empty array" that we normally refer tokmj
                                                       EmptyPartition<Product>   // it implements IEnumerable<T>, and provide quick handle for no elements e.g MoveNext() => false
                                                   }}
*/
//----------------Ʌ
```

Note that once you use `into`, the inner range variable goes out of the scope (outter range variable still valid):

```C#
// code doesn't compile
var q = from c in categories
        join p in products on c.IdCategory equals p.IdCategory into _
        select new {
           CategoryID = c.IdCategory,
           CategoryName = c.Name,
           Product = p.Description   // cannot access `p` any more, and if you think about it, it is like the groupby in SQL, that you cannot access a non aggregated column
        };

// as long as you use into, it get translated into GroupJoin call,  even when you are not accessing the new temp variable, 
var q = from c in categories()
        join p in products on c.IdCategory equals p.IdCategory into _
        select c;

var q = categories.GroupJoin(c => c.IdCategory, p => p.IdCategory, (c, _) => c);
```

## From Inner Join to Outer Join

There are two inner joins form in Linq:

```C#
// standard inner join
var query1 =
    from person in people
    join pet in pets on person equals pet.Owner
    select new
    {
        OwnerName = person.FirstName,
        PetName = pet.Name
    };

// you can also use GroupJoin to performance an inner join:
var query2 =
    from person in people
    join pet in pets on person equals pet.Owner into gj
    from subpet in gj
    select new
    {
        OwnerName = person.FirstName,
        PetName = subpet.Name
    };

//record class Person(string FirstName, string LastName);
//record class Pet(string Name, Person Owner);
```

you might ask why use `GroupJoin` (join into) to perform inner join? well, it is prerequiste to make 100% outer join combined with `DefaultIfEmpty()`:

```C#
var query =
    from person in people
    join pet in pets on person equals pet.Owner into gj
    from subpet in gj.DefaultIfEmpty()
    select new
    {
        person.FirstName,
        PetName = subpet?.Name ?? string.Empty
    };
```


## `let` Clause

Let's first see an intuitive example:

```C#
var q = from x in employees
        let tax = x.ComputeTax()
        orderby tax descending
        select x.LastName + ": " + tax   // x is still in scope, as you will see shortly, this x is not the same one as the first x
```
`let` is a little bit differnt than other operators like `into` which disables the scope of original scope variable, so after a `let` clause both the original range variable and the new one are in scope for the rest of the query.

The problem is, both "x" and "tax" are in scope at the same time… so what are we going to pass to the `Select` method at the end? We need one entity to pass through our query, which knows the value of both "x" and "tax" at any point, You can think of the above query as being translated into this:

```C#
var q = from x in employees
        select new { x, tax = x.ComputeTax() } into z
        orderby z.tax descending
        select z.x.LastName + ": " + z.tax

employees.Select(x => new { x, tax = x.ComputeTax() })
         .OrderByDescending(z => z.tax)
         .Select(z => z.x.LastName + ": " + z.tax);
```

**so you can think that `into` always project outer range variable**

Now you know the idea that how to keep alive of range variable's scope, let's see another exampl in multi-join with extending(or faking) range variable's scope:

```C#
Person p1 = new Person("Magnus");
// ...
List<Person> people = new() { p1, p2, p3 };

List<Cat> cats = new() { new(Name: "Barley", Owner: p1), new("Boots", Owner: p3) };

List<Dog> dogs = new() { new(Name: "Duke", Owner: p1), new("Denim", p2), new("Wiley", Owner: p3) };

var q = from person in people
        join cat in cats on person equals cat.Owner  // first note the join key doesn't need to be primitve type, it can be reference type
        join dog in dogs on person equals dog.Owner  // the person range variable in the second join is actlly lifted as anon.person
        select new {
           person,
           cat,
           dog
        };

var q = people.Join(cats, person => person, cat => cat.Owner, (person, cat) => new { person, cat })
              .Join(dogs, anon => anon.person, dog => dog.Owner, (anon, dog) => new { anon.person, anon.cat, dog });
```


## Aggregate

Aggregate includes `Sum`, `Max`, `Min` etc and `Aggregate`. Standard aggregate operators like `Sum`, `Max` etc are very simple, let's look at how to use `Aggregate`

```C#
Customer[] customers = GetCustomers();  // new Customer {Name = "Paolo", Orders = new Order[] {  new Order { IdOrder = 10010, Quantity = 3, IdProduct = 1 }} ...
Product[] products = GetProducts();     // new Product {IdProduct = 1, Price = 10 }

// extract the most expensive order for each custome
var q = from c in customers
        join o in (
             from c in customers
             from o in c.Orders
             join p in products on o.IdProduct equals p.IdProduct
             select new { c.Name, o.IdProduct, OrderAmount = o.Quantity * p.Price }
             ) on c.Name equals o.Name into orders
        select new { c.Name, MaxOrderAmount = orders.Aggregate((curr, next) => curr.OrderAmount > next.OrderAmount ? curr : next).OrderAmount };

/*
{ Name = Paolo, MaxOrderAmount = 100 }
{ Name = Marco, MaxOrderAmount = 600 }
{ Name = James, MaxOrderAmount = 600 }
{ Name = Frank, MaxOrderAmount = 1000 }
*/


// calculates the total amount ordered for each product
 var q = from p in products
         join o in (
              from c in customers
              from o in c.Orders
              join p in products on o.IdProduct equals p.IdProduct
              select new { c.Name, o.IdProduct, OrderAmount = o.Quantity * p.Price }
         ) on p.IdProduct equals o.IdProduct into productsGroup              //it's (accum, curr) that fits the context, not (accum, next), check the source code above you'll see
         select new { p.IdProduct, TotalOrderedAmount = productsGroup.Aggregate(0m, (accum, curr) => accum + curr.OrderAmount) };

/*
{ IdProduct = 1, TotalOrderedAmount = 130 }
{ IdProduct = 2, TotalOrderedAmount = 100 }
{ IdProduct = 3, TotalOrderedAmount = 1200 }
{ IdProduct = 4, TotalOrderedAmount = 0 }
{ IdProduct = 5, TotalOrderedAmount = 1000 }
{ IdProduct = 6, TotalOrderedAmount = 0 }
*/
```

## Other Operators

```C#
var list = new int[] { 1, 2, 3, 4, 5, -1, -2 };

var q = list.Where(x => x <= 3);       // 1, 2, 3, -1, -2

var q = list.TakeWhile(x => x <= 3);   // 1, 2, 3

var q = list.SkipWhile(x => x <= 3);   // 4, 5, -1, -2
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
