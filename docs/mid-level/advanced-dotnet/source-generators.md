# Source Generators

## Giriş

Source Generators, .NET 5+ ile gelen compile-time code generation teknolojisidir. Mid-level geliştiriciler için source generators'ı anlamak, performance optimization, boilerplate code reduction ve compile-time validation için kritik öneme sahiptir. Bu dosya, source generator fundamentals, custom generators, incremental generators ve best practices konularını kapsar.

## Source Generator Fundamentals

### 1. Basic Source Generator
Temel source generator implementasyonu.

```csharp
[Generator]
public class HelloWorldGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
        // Register for syntax notifications
        context.RegisterForSyntaxNotifications(() => new HelloWorldSyntaxReceiver());
    }

    public void Execute(GeneratorExecutionContext context)
    {
        // Get the registered receiver
        if (context.SyntaxReceiver is not HelloWorldSyntaxReceiver receiver)
            return;

        // Generate source code
        var sourceBuilder = new StringBuilder(@"
using System;

namespace Generated
{
    public static class HelloWorld
    {
        public static void SayHello()
        {
            Console.WriteLine(""Hello from Source Generator!"");
        }
    }
}");

        // Add the generated source
        context.AddSource("HelloWorld.g.cs", SourceText.From(sourceBuilder.ToString(), Encoding.UTF8));
    }
}

public class HelloWorldSyntaxReceiver : ISyntaxReceiver
{
    public List<ClassDeclarationSyntax> Classes { get; } = new();

    public void OnVisitSyntaxNode(SyntaxNode syntaxNode)
    {
        if (syntaxNode is ClassDeclarationSyntax classDeclaration)
        {
            Classes.Add(classDeclaration);
        }
    }
}
```

### 2. Attribute-Based Generator
Attribute kullanarak source generation tetikleyen generator.

```csharp
[Generator]
public class AutoToStringGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
        context.RegisterForSyntaxNotifications(() => new AutoToStringSyntaxReceiver());
    }

    public void Execute(GeneratorExecutionContext context)
    {
        if (context.SyntaxReceiver is not AutoToStringSyntaxReceiver receiver)
            return;

        foreach (var classDeclaration in receiver.Classes)
        {
            var className = classDeclaration.Identifier.ValueText;
            var properties = GetProperties(classDeclaration);
            
            var source = GenerateToStringMethod(className, properties);
            context.AddSource($"{className}.ToString.g.cs", SourceText.From(source, Encoding.UTF8));
        }
    }

    private string GenerateToStringMethod(string className, List<PropertyInfo> properties)
    {
        var propertyStrings = properties.Select(p => $"{p.Name} = {{{p.Name}}}");
        var toStringBody = string.Join(", ", propertyStrings);
        
        return $@"
using System;

namespace Generated
{{
    public partial class {className}
    {{
        public override string ToString()
        {{
            return $""{className} {{{toStringBody}}}"";
        }}
    }}
}}";
    }

    private List<PropertyInfo> GetProperties(ClassDeclarationSyntax classDeclaration)
    {
        return classDeclaration.Members
            .OfType<PropertyDeclarationSyntax>()
            .Select(p => new PropertyInfo { Name = p.Identifier.ValueText })
            .ToList();
    }
}

public class PropertyInfo
{
    public string Name { get; set; }
}

public class AutoToStringSyntaxReceiver : ISyntaxReceiver
{
    public List<ClassDeclarationSyntax> Classes { get; } = new();

    public void OnVisitSyntaxNode(SyntaxNode syntaxNode)
    {
        if (syntaxNode is ClassDeclarationSyntax classDeclaration &&
            HasAutoToStringAttribute(classDeclaration))
        {
            Classes.Add(classDeclaration);
        }
    }

    private bool HasAutoToStringAttribute(ClassDeclarationSyntax classDeclaration)
    {
        return classDeclaration.AttributeLists
            .SelectMany(al => al.Attributes)
            .Any(a => a.Name.ToString() == "AutoToString");
    }
}

[AttributeUsage(AttributeTargets.Class)]
public class AutoToStringAttribute : Attribute
{
}
```

## Incremental Source Generators

### 1. Performance-Optimized Generator
Incremental generation kullanan generator.

```csharp
[Generator]
public class IncrementalGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // Create a pipeline for class declarations
        var classDeclarations = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: static (s, _) => IsSyntaxTargetForGeneration(s),
                transform: static (ctx, _) => GetTargetForGeneration(ctx))
            .Where(static m => m is not null);

        // Generate the source
        context.RegisterSourceOutput(classDeclarations,
            static (spc, source) => Execute(source!, spc));
    }

    private static bool IsSyntaxTargetForGeneration(SyntaxNode node)
        => node is ClassDeclarationSyntax { AttributeLists.Count: > 0 };

    private static ClassDeclarationSyntax? GetTargetForGeneration(GeneratorSyntaxContext context)
    {
        var classDeclarationSyntax = (ClassDeclarationSyntax)context.Node;
        
        foreach (AttributeListSyntax attributeListSyntax in classDeclarationSyntax.AttributeLists)
        {
            foreach (AttributeSyntax attributeSyntax in attributeListSyntax.Attributes)
            {
                if (IsAutoToStringAttribute(attributeSyntax))
                {
                    return classDeclarationSyntax;
                }
            }
        }
        
        return null;
    }

    private static bool IsAutoToStringAttribute(AttributeSyntax attributeSyntax)
    {
        var name = attributeSyntax.Name.ToString();
        return name == "AutoToString" || name == "AutoToStringAttribute";
    }

    private static void Execute(ClassDeclarationSyntax classDeclaration, SourceProductionContext context)
    {
        var className = classDeclaration.Identifier.ValueText;
        var namespaceName = GetNamespace(classDeclaration);
        
        var source = GenerateSource(className, namespaceName);
        context.AddSource($"{className}.g.cs", SourceText.From(source, Encoding.UTF8));
    }

    private static string GetNamespace(ClassDeclarationSyntax classDeclaration)
    {
        var parent = classDeclaration.Parent;
        while (parent is not NamespaceDeclarationSyntax namespaceDeclaration)
        {
            parent = parent?.Parent;
        }
        return namespaceDeclaration?.Name.ToString() ?? "Generated";
    }

    private static string GenerateSource(string className, string namespaceName)
    {
        return $@"
using System;

namespace {namespaceName}
{{
    public partial class {className}
    {{
        public string GeneratedProperty {{ get; set; }} = ""Generated by Source Generator"";
        
        public void GeneratedMethod()
        {{
            Console.WriteLine(""This method was generated at compile time"");
        }}
    }}
}}";
    }
}
```

## Advanced Source Generators

### 1. JSON Serialization Generator
JSON serialization için custom generator.

```csharp
[Generator]
public class JsonSerializerGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var classDeclarations = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: static (s, _) => s is ClassDeclarationSyntax,
                transform: static (ctx, _) => ctx.Node as ClassDeclarationSyntax)
            .Where(static m => m is not null);

        context.RegisterSourceOutput(classDeclarations,
            static (spc, source) => GenerateJsonSerializer(source!, spc));
    }

    private static void GenerateJsonSerializer(ClassDeclarationSyntax classDeclaration, SourceProductionContext context)
    {
        var className = classDeclaration.Identifier.ValueText;
        var properties = GetSerializableProperties(classDeclaration);
        
        if (!properties.Any()) return;
        
        var source = GenerateJsonSerializerSource(className, properties);
        context.AddSource($"{className}.JsonSerializer.g.cs", SourceText.From(source, Encoding.UTF8));
    }

    private static List<PropertyInfo> GetSerializableProperties(ClassDeclarationSyntax classDeclaration)
    {
        return classDeclaration.Members
            .OfType<PropertyDeclarationSyntax>()
            .Where(p => !HasIgnoreAttribute(p))
            .Select(p => new PropertyInfo 
            { 
                Name = p.Identifier.ValueText,
                Type = p.Type?.ToString() ?? "object"
            })
            .ToList();
    }

    private static bool HasIgnoreAttribute(PropertyDeclarationSyntax property)
    {
        return property.AttributeLists
            .SelectMany(al => al.Attributes)
            .Any(a => a.Name.ToString() == "JsonIgnore");
    }

    private static string GenerateJsonSerializerSource(string className, List<PropertyInfo> properties)
    {
        var serializationCode = string.Join("\n            ", 
            properties.Select(p => $"\"{p.Name}\": {GetJsonValue(p)}"));
        
        var deserializationCode = string.Join("\n            ", 
            properties.Select(p => $"{p.Name} = json[\"{p.Name}\"].{GetDeserializationMethod(p.Type)}"));
        
        return $@"
using System;
using System.Text.Json;
using System.Collections.Generic;

namespace Generated
{{
    public partial class {className}
    {{
        public string ToJson()
        {{
            return JsonSerializer.Serialize(new
            {{
                {serializationCode}
            }});
        }}
        
        public static {className} FromJson(string json)
        {{
            var jsonDoc = JsonDocument.Parse(json);
            var root = jsonDoc.RootElement;
            
            return new {className}
            {{
                {deserializationCode}
            }};
        }}
    }}
}}";
    }

    private static string GetJsonValue(PropertyInfo property)
    {
        return property.Type switch
        {
            "string" => $"\"{property.Name}\"",
            "int" or "long" or "double" or "decimal" => property.Name,
            "bool" => property.Name,
            _ => $"JsonSerializer.Serialize({property.Name})"
        };
    }

    private static string GetDeserializationMethod(string type)
    {
        return type switch
        {
            "string" => "GetString()",
            "int" => "GetInt32()",
            "long" => "GetInt64()",
            "double" => "GetDouble()",
            "decimal" => "GetDecimal()",
            "bool" => "GetBoolean()",
            _ => "GetString()"
        };
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class JsonIgnoreAttribute : Attribute
{
}
```

## Mülakat Soruları

### Temel Sorular

1. **Source Generators nedir ve neden kullanılır?**
   - **Cevap**: Compile-time code generation teknolojisi. Performance improvement, boilerplate reduction, compile-time validation için.

2. **Source Generators vs Reflection farkı nedir?**
   - **Cevap**: Source generators compile-time, reflection runtime. Source generators daha performanslı, type-safe.

3. **Incremental Generators nedir?**
   - **Cevap**: Performance-optimized generators, sadece değişen kısımları regenerate eder.

4. **Source Generators ne zaman kullanılır?**
   - **Cevap**: Boilerplate code, compile-time validation, performance-critical scenarios.

5. **Source Generators limitations nelerdir?**
   - **Cevap**: Compile-time only, no runtime access, limited debugging.

### Teknik Sorular

1. **Custom Source Generator nasıl implement edilir?**
   - **Cevap**: ISourceGenerator interface, Initialize ve Execute methods, syntax analysis.

2. **Incremental Generator nasıl optimize edilir?**
   - **Cevap**: IIncrementalGenerator interface, syntax provider pipeline, conditional generation.

3. **Source Generator debugging nasıl yapılır?**
   - **Cevap**: Debugger.Launch(), conditional compilation, logging.

4. **Source Generator testing nasıl yapılır?**
   - **Cevap**: SourceGeneratorVerifier, expected output validation, unit testing.

5. **Source Generator performance nasıl ölçülür?**
   - **Cevap**: Compilation time measurement, memory usage, incremental generation efficiency.

## Best Practices

1. **Generator Design**
   - Single responsibility principle uygulayın
   - Performance optimize edin
   - Error handling implement edin
   - Documentation sağlayın

2. **Performance Optimization**
   - Incremental generation kullanın
   - Conditional compilation implement edin
   - Memory allocation minimize edin
   - Caching strategies uygulayın

3. **Error Handling**
   - Compilation errors handle edin
   - User-friendly error messages sağlayın
   - Fallback mechanisms implement edin
   - Validation logic ekleyin

4. **Testing & Debugging**
   - Unit tests yazın
   - Integration tests implement edin
   - Debugging tools ekleyin
   - Performance testing yapın

5. **Documentation**
   - Usage examples sağlayın
   - API documentation yazın
   - Best practices document edin
   - Troubleshooting guide oluşturun

## Kaynaklar

- [Source Generators](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview)
- [Incremental Generators](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md)
- [Source Generator Cookbook](https://github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md)
- [Performance Best Practices](https://github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md#performance-considerations)
- [Testing Source Generators](https://github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md#unit-testing-of-generators)
