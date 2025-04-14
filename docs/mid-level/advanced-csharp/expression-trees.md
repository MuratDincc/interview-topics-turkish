# Expression Trees

## Genel Bakış
Expression Trees (İfade Ağaçları), C#'ta kodun yapısını temsil eden ve çalışma zamanında analiz edilebilen, değiştirilebilen ve çalıştırılabilen veri yapılarıdır. LINQ sorguları, dinamik sorgular ve kod üretimi gibi senaryolarda kullanılır.

## Mülakat Soruları ve Cevapları

### 1. Expression Trees nedir ve ne için kullanılır?
**Cevap:**
Expression Trees kullanım senaryoları:
- LINQ sorgularının çevirisi
- Dinamik sorgu oluşturma
- Kod üretimi
- Metod çağrılarının analizi
- Validation kuralları

**Örnek Kod:**
```csharp
// Basit expression tree örneği
Expression<Func<int, int, int>> add = (a, b) => a + b;

// Expression tree oluşturma
ParameterExpression paramA = Expression.Parameter(typeof(int), "a");
ParameterExpression paramB = Expression.Parameter(typeof(int), "b");
BinaryExpression addExpr = Expression.Add(paramA, paramB);
Expression<Func<int, int, int>> addLambda = 
    Expression.Lambda<Func<int, int, int>>(addExpr, paramA, paramB);

// Expression tree çalıştırma
Func<int, int, int> addFunc = addLambda.Compile();
int result = addFunc(5, 3); // Sonuç: 8
```

### 2. Expression Trees nasıl oluşturulur ve manipüle edilir?
**Cevap:**
Expression Trees manipülasyonu:
- Expression sınıfları
- ParameterExpression
- BinaryExpression
- MethodCallExpression
- LambdaExpression

**Örnek Kod:**
```csharp
// Expression tree manipülasyonu
public class ExpressionBuilder
{
    public Expression<Func<T, bool>> CreateGreaterThanExpression<T>(
        string propertyName, object value)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        var property = Expression.Property(parameter, propertyName);
        var constant = Expression.Constant(value);
        var greaterThan = Expression.GreaterThan(property, constant);
        
        return Expression.Lambda<Func<T, bool>>(greaterThan, parameter);
    }

    public Expression<Func<T, object>> CreatePropertySelector<T>(
        string propertyName)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        var property = Expression.Property(parameter, propertyName);
        var convert = Expression.Convert(property, typeof(object));
        
        return Expression.Lambda<Func<T, object>>(convert, parameter);
    }
}

// Kullanım örneği
var builder = new ExpressionBuilder();
var greaterThanExpr = builder.CreateGreaterThanExpression<Person>("Age", 18);
var propertySelector = builder.CreatePropertySelector<Person>("Name");
```

### 3. Expression Trees ile dinamik sorgular nasıl oluşturulur?
**Cevap:**
Dinamik sorgu oluşturma:
- Filter oluşturma
- Order by oluşturma
- Select oluşturma
- Join oluşturma

**Örnek Kod:**
```csharp
// Dinamik sorgu oluşturma
public class DynamicQueryBuilder
{
    public IQueryable<T> ApplyFilter<T>(
        IQueryable<T> query,
        string propertyName,
        object value,
        string @operator)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        var property = Expression.Property(parameter, propertyName);
        var constant = Expression.Constant(value);
        
        BinaryExpression binaryExpression = @operator switch
        {
            "==" => Expression.Equal(property, constant),
            ">" => Expression.GreaterThan(property, constant),
            "<" => Expression.LessThan(property, constant),
            ">=" => Expression.GreaterThanOrEqual(property, constant),
            "<=" => Expression.LessThanOrEqual(property, constant),
            _ => throw new ArgumentException("Geçersiz operatör")
        };
        
        var lambda = Expression.Lambda<Func<T, bool>>(binaryExpression, parameter);
        return query.Where(lambda);
    }

    public IQueryable<T> ApplyOrderBy<T>(
        IQueryable<T> query,
        string propertyName,
        bool descending = false)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        var property = Expression.Property(parameter, propertyName);
        var lambda = Expression.Lambda(property, parameter);
        
        string methodName = descending ? "OrderByDescending" : "OrderBy";
        var resultExpression = Expression.Call(
            typeof(Queryable),
            methodName,
            new Type[] { typeof(T), property.Type },
            query.Expression,
            Expression.Quote(lambda));
            
        return query.Provider.CreateQuery<T>(resultExpression);
    }
}
```

### 4. Expression Trees performans optimizasyonu nasıl yapılır?
**Cevap:**
Performans optimizasyonu için:
- Expression caching
- Compile optimizasyonu
- Expression tree basitleştirme
- Memory management

**Örnek Kod:**
```csharp
// Expression optimizasyonu
public class ExpressionCache
{
    private readonly Dictionary<string, Delegate> _cache = new();

    public TDelegate GetOrCreate<TDelegate>(
        string key,
        Func<Expression<TDelegate>> expressionFactory)
        where TDelegate : Delegate
    {
        if (!_cache.TryGetValue(key, out var compiledDelegate))
        {
            var expression = expressionFactory();
            compiledDelegate = expression.Compile();
            _cache[key] = compiledDelegate;
        }
        
        return (TDelegate)compiledDelegate;
    }
}

// Expression basitleştirme
public class ExpressionSimplifier
{
    public Expression Simplify(Expression expression)
    {
        return new Simplifier().Visit(expression);
    }

    private class Simplifier : ExpressionVisitor
    {
        protected override Expression VisitBinary(BinaryExpression node)
        {
            // Sabit ifadeleri hesapla
            if (node.NodeType == ExpressionType.Add &&
                node.Left is ConstantExpression left &&
                node.Right is ConstantExpression right)
            {
                return Expression.Constant(
                    (int)left.Value + (int)right.Value);
            }
            
            return base.VisitBinary(node);
        }
    }
}
```

### 5. Expression Trees ile kod üretimi nasıl yapılır?
**Cevap:**
Kod üretimi için:
- DynamicMethod
- ILGenerator
- Expression tree dönüşümü
- Runtime kod üretimi

**Örnek Kod:**
```csharp
// Kod üretimi örneği
public class CodeGenerator
{
    public Func<T, TResult> GenerateGetter<T, TResult>(
        string propertyName)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        var property = Expression.Property(parameter, propertyName);
        var lambda = Expression.Lambda<Func<T, TResult>>(property, parameter);
        
        return lambda.Compile();
    }

    public Action<T, TValue> GenerateSetter<T, TValue>(
        string propertyName)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        var value = Expression.Parameter(typeof(TValue), "value");
        var property = Expression.Property(parameter, propertyName);
        var assign = Expression.Assign(property, value);
        var lambda = Expression.Lambda<Action<T, TValue>>(assign, parameter, value);
        
        return lambda.Compile();
    }
}

// Kullanım örneği
var generator = new CodeGenerator();
var getter = generator.GenerateGetter<Person, string>("Name");
var setter = generator.GenerateSetter<Person, string>("Name");

var person = new Person();
setter(person, "John");
string name = getter(person); // "John"
```

## Best Practices
1. **Kullanım**
   - Uygun expression tree'leri seçin
   - Expression'ları doğru yerde kullanın
   - Documentation ekleyin
   - Error handling yapın

2. **Performans**
   - Expression caching kullanın
   - Compile optimizasyonu yapın
   - Expression tree'leri basitleştirin
   - Memory management yapın

3. **Bakım**
   - Kod okunabilirliğini koruyun
   - Test coverage sağlayın
   - Version control yapın
   - Documentation güncelleyin

## Kaynaklar
- [Microsoft Expression Trees](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/concepts/expression-trees/)
- [Expression Tree Building](https://docs.microsoft.com/tr-tr/dotnet/api/system.linq.expressions)
- [Dynamic LINQ](https://docs.microsoft.com/tr-tr/dotnet/api/system.linq.dynamic)
- [Expression Tree Performance](https://docs.microsoft.com/tr-tr/dotnet/framework/reflection-and-codedom/how-to-use-expression-trees-to-build-dynamic-queries) 