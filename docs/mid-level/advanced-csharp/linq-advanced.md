# LINQ Advanced

## Genel Bakış
LINQ (Language Integrated Query), C#'ta veri sorgulama için kullanılan güçlü bir özelliktir. İleri seviye LINQ kullanımı, karmaşık veri işlemlerini daha verimli ve okunabilir hale getirir.

## Mülakat Soruları ve Cevapları

### 1. LINQ Query Syntax vs Method Syntax arasındaki farklar nelerdir?
**Cevap:**
Query Syntax ve Method Syntax farkları:
- Okunabilirlik
- Performans
- Kullanım senaryoları
- Compile-time checking

**Örnek Kod:**
```csharp
// Query Syntax
var querySyntax = from student in students
                 where student.Age > 18
                 orderby student.Name
                 select new { student.Name, student.Age };

// Method Syntax
var methodSyntax = students
    .Where(student => student.Age > 18)
    .OrderBy(student => student.Name)
    .Select(student => new { student.Name, student.Age });

// Karmaşık sorgu örneği
var complexQuery = from student in students
                  join course in courses on student.Id equals course.StudentId
                  where student.Age > 18 && course.Grade > 80
                  group student by student.Department into deptGroup
                  select new
                  {
                      Department = deptGroup.Key,
                      AverageGrade = deptGroup.Average(s => s.Grades.Average())
                  };
```

### 2. LINQ'da lazy evaluation nasıl çalışır?
**Cevap:**
Lazy evaluation özellikleri:
- Deferred execution
- Streaming
- Memory optimizasyonu
- Query composition

**Örnek Kod:**
```csharp
// Lazy evaluation örneği
public IEnumerable<int> GetNumbers()
{
    var numbers = Enumerable.Range(1, 1000000);
    
    // Sorgu henüz çalıştırılmadı
    var query = numbers.Where(n => n % 2 == 0)
                      .Select(n => n * 2);
    
    // Sorgu çalıştırıldı
    foreach (var number in query)
    {
        yield return number;
    }
}

// Immediate execution örneği
public List<int> GetNumbersImmediate()
{
    var numbers = Enumerable.Range(1, 1000000);
    
    // Sorgu hemen çalıştırıldı
    return numbers.Where(n => n % 2 == 0)
                 .Select(n => n * 2)
                 .ToList();
}
```

### 3. Custom LINQ extension metodları nasıl yazılır?
**Cevap:**
Custom extension metodları için:
- Extension method syntax
- Generic type parameters
- Lambda expressions
- Query composition

**Örnek Kod:**
```csharp
// Custom extension metod örneği
public static class LinqExtensions
{
    public static IEnumerable<T> WhereIf<T>(
        this IEnumerable<T> source,
        bool condition,
        Func<T, bool> predicate)
    {
        return condition ? source.Where(predicate) : source;
    }

    public static IEnumerable<T> DistinctBy<T, TKey>(
        this IEnumerable<T> source,
        Func<T, TKey> keySelector)
    {
        var seenKeys = new HashSet<TKey>();
        return source.Where(item => seenKeys.Add(keySelector(item)));
    }

    public static IEnumerable<T> Paginate<T>(
        this IEnumerable<T> source,
        int pageNumber,
        int pageSize)
    {
        return source.Skip((pageNumber - 1) * pageSize)
                    .Take(pageSize);
    }
}

// Kullanım örneği
var filteredData = data
    .WhereIf(shouldFilter, x => x.IsActive)
    .DistinctBy(x => x.Id)
    .Paginate(pageNumber: 1, pageSize: 10);
```

### 4. LINQ performans optimizasyonu nasıl yapılır?
**Cevap:**
Performans optimizasyonu için:
- Index kullanımı
- Caching
- Query optimization
- Materialization stratejileri

**Örnek Kod:**
```csharp
// Performans optimizasyonu örnekleri
public class LinqOptimization
{
    // Index kullanımı
    public IEnumerable<Student> GetStudentsByIndex(List<Student> students)
    {
        var index = students.ToDictionary(s => s.Id);
        return students.Where(s => index.ContainsKey(s.Id));
    }

    // Caching örneği
    private Dictionary<int, Student> _studentCache;
    public Student GetStudentWithCache(int id)
    {
        return _studentCache.GetOrAdd(id, 
            key => students.FirstOrDefault(s => s.Id == key));
    }

    // Query optimization
    public IQueryable<Student> OptimizeQuery(IQueryable<Student> query)
    {
        return query.Where(s => s.IsActive)
                   .OrderBy(s => s.Name)
                   .Take(100)
                   .AsNoTracking();
    }
}
```

### 5. Expression Trees nedir ve nasıl kullanılır?
**Cevap:**
Expression Trees özellikleri:
- Runtime query generation
- Dynamic query building
- Query translation
- Custom provider support

**Örnek Kod:**
```csharp
// Expression Tree örneği
public class ExpressionBuilder
{
    public Expression<Func<T, bool>> BuildPredicate<T>(
        string propertyName,
        string value)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        var property = Expression.Property(parameter, propertyName);
        var constant = Expression.Constant(value);
        var body = Expression.Equal(property, constant);
        
        return Expression.Lambda<Func<T, bool>>(body, parameter);
    }

    // Dynamic query oluşturma
    public IQueryable<T> ApplyDynamicFilter<T>(
        IQueryable<T> source,
        string propertyName,
        string value)
    {
        var predicate = BuildPredicate<T>(propertyName, value);
        return source.Where(predicate);
    }
}

// Kullanım örneği
var query = db.Students;
var filteredQuery = expressionBuilder
    .ApplyDynamicFilter(query, "Name", "John");
```

## Best Practices
1. **Performans**
   - Lazy evaluation kullanın
   - Gereksiz materialization'dan kaçının
   - Index'leri etkin kullanın
   - Query'leri optimize edin

2. **Kod Kalitesi**
   - Query syntax'ı tercih edin
   - Extension metodları kullanın
   - Lambda ifadelerini düzenli kullanın
   - Documentation ekleyin

3. **Bakım**
   - Karmaşık sorguları parçalayın
   - Reusable component'ler oluşturun
   - Test edilebilir kod yazın
   - Error handling ekleyin

## Kaynaklar
- [Microsoft LINQ Documentation](https://docs.microsoft.com/tr-tr/dotnet/csharp/programming-guide/concepts/linq/)
- [LINQ Performance](https://docs.microsoft.com/tr-tr/dotnet/standard/linq/performance)
- [Expression Trees](https://docs.microsoft.com/tr-tr/dotnet/csharp/expression-trees)
- [LINQ Best Practices](https://docs.microsoft.com/tr-tr/dotnet/standard/linq/write-linq-queries) 