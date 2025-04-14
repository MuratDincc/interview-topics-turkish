# Reflection

## Genel Bakış

Reflection, .NET'te çalışma zamanında (runtime) tip bilgilerini inceleme, tip örnekleri oluşturma ve dinamik olarak metod çağırma gibi işlemleri yapmamızı sağlayan güçlü bir özelliktir. Bu özellik sayesinde, derleme zamanında bilinmeyen tipleri ve üyelerini çalışma zamanında keşfedebilir ve kullanabiliriz.

## Temel Kullanım Alanları

1. **Tip Bilgisi İnceleme**
   - Bir tipin özelliklerini, metodlarını ve alanlarını keşfetme
   - Tip hiyerarşisini inceleme
   - Attribute'ları okuma

2. **Dinamik Tip Oluşturma**
   - Çalışma zamanında yeni tip örnekleri oluşturma
   - Interface'leri dinamik olarak implemente etme
   - Proxy nesneleri oluşturma

3. **Dinamik Metod Çağırma**
   - Metodları isimlerine göre çağırma
   - Private üyelere erişim
   - Generic metodları dinamik olarak çağırma

## Örnek Kullanımlar

### 1. Tip Bilgisi İnceleme

```csharp
Type type = typeof(MyClass);

// Özellikleri listeleme
foreach (PropertyInfo prop in type.GetProperties())
{
    Console.WriteLine($"Property: {prop.Name}, Type: {prop.PropertyType}");
}

// Metodları listeleme
foreach (MethodInfo method in type.GetMethods())
{
    Console.WriteLine($"Method: {method.Name}, Return Type: {method.ReturnType}");
}

// Attribute'ları okuma
foreach (Attribute attr in type.GetCustomAttributes())
{
    Console.WriteLine($"Attribute: {attr.GetType().Name}");
}
```

### 2. Dinamik Tip Oluşturma

```csharp
// Assembly'den tip yükleme
Assembly assembly = Assembly.Load("MyAssembly");
Type type = assembly.GetType("MyNamespace.MyClass");

// Tip örneği oluşturma
object instance = Activator.CreateInstance(type);

// Generic tip oluşturma
Type genericType = typeof(List<>);
Type constructedType = genericType.MakeGenericType(typeof(int));
object list = Activator.CreateInstance(constructedType);
```

### 3. Dinamik Metod Çağırma

```csharp
// Metod bilgisini alma
MethodInfo method = type.GetMethod("MyMethod");

// Metodu çağırma
object result = method.Invoke(instance, new object[] { "param1", 42 });

// Generic metod çağırma
MethodInfo genericMethod = type.GetMethod("GenericMethod");
MethodInfo constructedMethod = genericMethod.MakeGenericMethod(typeof(string));
constructedMethod.Invoke(instance, new object[] { "value" });
```

## Dikkat Edilmesi Gerekenler

1. **Performans**
   - Reflection işlemleri normal kod çağrılarına göre daha yavaştır
   - Sık kullanılan reflection işlemleri için cache mekanizması kullanılmalıdır

2. **Güvenlik**
   - Private üyelere erişim sağlayabildiği için dikkatli kullanılmalıdır
   - Güvenlik kontrolleri yapılmalıdır

3. **Bakım**
   - Reflection kullanılan kodlar daha zor debug edilebilir
   - Refactoring işlemleri daha zor olabilir

## Best Practices

1. **Cache Kullanımı**
   ```csharp
   private static readonly Dictionary<string, MethodInfo> _methodCache = new();
   
   public MethodInfo GetCachedMethod(string methodName)
   {
       if (!_methodCache.TryGetValue(methodName, out MethodInfo method))
       {
           method = typeof(MyClass).GetMethod(methodName);
           _methodCache[methodName] = method;
       }
       return method;
   }
   ```

2. **Tip Güvenliği**
   ```csharp
   public T CreateInstance<T>(string typeName) where T : class
   {
       Type type = Type.GetType(typeName);
       if (type == null || !typeof(T).IsAssignableFrom(type))
       {
           throw new ArgumentException($"Type {typeName} is not valid");
       }
       return (T)Activator.CreateInstance(type);
   }
   ```

3. **Hata Yönetimi**
   ```csharp
   try
   {
       MethodInfo method = type.GetMethod("MyMethod");
       if (method == null)
       {
           throw new MissingMethodException($"Method MyMethod not found in type {type.Name}");
       }
       return method.Invoke(instance, parameters);
   }
   catch (TargetInvocationException ex)
   {
       throw ex.InnerException ?? ex;
   }
   ```

## Kullanım Senaryoları

1. **Dependency Injection Framework'leri**
   - Tip bağımlılıklarını otomatik çözme
   - Constructor injection için reflection kullanımı

2. **ORM (Object-Relational Mapping)**
   - Entity property'lerini otomatik eşleştirme
   - Dinamik sorgu oluşturma

3. **Serialization/Deserialization**
   - JSON/XML dönüşümleri
   - Custom serializer implementasyonları

4. **Plugin Sistemleri**
   - Dinamik olarak yüklenen assembly'leri keşfetme
   - Interface implementasyonlarını bulma

5. **Unit Testing Framework'leri**
   - Test metodlarını otomatik bulma
   - Test fixture'ları oluşturma

## Performans İyileştirmeleri

1. **Expression Trees Kullanımı**
   ```csharp
   private static readonly Dictionary<string, Func<object, object>> _propertyGetters = new();
   
   public static Func<object, object> CreatePropertyGetter(PropertyInfo property)
   {
       var instance = Expression.Parameter(typeof(object), "instance");
       var cast = Expression.Convert(instance, property.DeclaringType);
       var propertyAccess = Expression.Property(cast, property);
       var convert = Expression.Convert(propertyAccess, typeof(object));
       return Expression.Lambda<Func<object, object>>(convert, instance).Compile();
   }
   ```

2. **Delegate Cache**
   ```csharp
   private static readonly ConcurrentDictionary<MethodInfo, Delegate> _methodCache = new();
   
   public static TDelegate CreateDelegate<TDelegate>(MethodInfo method)
       where TDelegate : Delegate
   {
       return (TDelegate)_methodCache.GetOrAdd(method, m => 
           Delegate.CreateDelegate(typeof(TDelegate), m));
   }
   ```

## Güvenlik Konuları

1. **Code Access Security (CAS)**
   - ReflectionPermission kontrolü
   - Kısıtlı reflection erişimi

2. **Private Member Erişimi**
   ```csharp
   public static T GetPrivateField<T>(object instance, string fieldName)
   {
       var field = instance.GetType().GetField(fieldName, 
           BindingFlags.Instance | BindingFlags.NonPublic);
       return (T)field?.GetValue(instance);
   }
   ```

3. **Assembly Yükleme Güvenliği**
   ```csharp
   public static Assembly LoadAssemblySafely(string assemblyPath)
   {
       try
       {
           return Assembly.LoadFrom(assemblyPath);
       }
       catch (FileLoadException)
       {
           // Güvenlik politikası ihlali
           throw new SecurityException("Assembly yüklenemedi");
       }
   }
   ```

## İleri Seviye Konular

1. **DynamicMethod ile IL Generation**
   ```csharp
   public static DynamicMethod CreateDynamicMethod()
   {
       var method = new DynamicMethod("DynamicMethod", 
           typeof(void), new[] { typeof(object) });
       var il = method.GetILGenerator();
       // IL kodları
       il.Emit(OpCodes.Ret);
       return method;
   }
   ```

2. **Custom Attribute Oluşturma**
   ```csharp
   [AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
   public class MyCustomAttribute : Attribute
   {
       public string Description { get; }
       
       public MyCustomAttribute(string description)
       {
           Description = description;
       }
   }
   ```

3. **Generic Tip Manipülasyonu**
   ```csharp
   public static Type MakeGenericType(Type genericType, params Type[] typeArguments)
   {
       if (!genericType.IsGenericTypeDefinition)
       {
           throw new ArgumentException("Type must be a generic type definition");
       }
       return genericType.MakeGenericType(typeArguments);
   }
   ```

## Kaynaklar

- [Microsoft Docs - Reflection](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/reflection)
- [C# Reflection Best Practices](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/security-considerations-for-reflection)
- [Performance Considerations for Reflection](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/security-considerations-for-reflection) 