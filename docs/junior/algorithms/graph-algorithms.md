# Graph Algoritmaları

## Genel Bakış

Graph (Graf) algoritmaları, düğümler (nodes/vertices) ve kenarlar (edges) ile oluşturulan veri yapıları üzerinde çalışan algoritmalardır. Sosyal ağlar, harita navigasyonu, bağımlılık yönetimi, ağ topolojisi gibi gerçek dünya problemlerinin büyük çoğunluğu graph yapıları ile modellenir.

**Temel Kavramlar:**

| Kavram | Açıklama |
|---|---|
| Vertex (Düğüm) | Graphtaki her bir nokta |
| Edge (Kenar) | İki düğüm arasındaki bağlantı |
| Directed (Yönlü) | Kenarların yönü vardır (A → B, B'den A'ya gidilemez) |
| Undirected (Yönsüz) | Kenarların yönü yoktur (A — B, her iki yönde gidilebilir) |
| Weighted (Ağırlıklı) | Kenarlarda maliyet/mesafe bilgisi vardır |
| Unweighted (Ağırlıksız) | Tüm kenarlar eşit ağırlıktadır |
| Degree | Bir düğüme bağlı kenar sayısı |
| Path | Bir düğümden diğerine giden kenar dizisi |
| Cycle | Başlangıç düğümüne geri dönen path |
| DAG | Directed Acyclic Graph — yönlü, döngüsüz graph |

**Adjacency List vs Adjacency Matrix:**

| Özellik | Adjacency List | Adjacency Matrix |
|---|---|---|
| Alan Karmaşıklığı | O(V + E) | O(V²) |
| Kenar varlığı kontrolü | O(degree) | O(1) |
| Tüm komşuları listeleme | O(degree) | O(V) |
| Seyrek graphlar için | Daha verimli | Verimsiz |
| Yoğun graphlar için | Kabul edilebilir | Daha verimli |

Çoğu pratik uygulamada graphlar seyrektir (E << V²), bu yüzden **Adjacency List** tercih edilir.

---

## Mülakat Soruları ve Cevapları

### 1. Soru

**C# ile bir Graph veri yapısını nasıl temsil edersiniz? Adjacency List yaklaşımını generic bir sınıf olarak implemente edin.**

**Cevap:**

Graph'ı C#'ta temsil etmenin en yaygın yolu `Dictionary<T, List<T>>` kullanmaktır. Bu yapı, her düğümün komşularını dinamik bir liste olarak tutar. Generic bir Graph sınıfı, hem yönlü hem de yönsüz graphları destekleyecek şekilde tasarlanabilir.

**Örnek Kod:**

```csharp
public class Graph<T> where T : notnull
{
    private readonly Dictionary<T, List<T>> _adjacencyList;
    private readonly bool _isDirected;

    public Graph(bool isDirected = false)
    {
        _adjacencyList = new Dictionary<T, List<T>>();
        _isDirected = isDirected;
    }

    // Düğüm ekleme
    public void AddVertex(T vertex)
    {
        if (!_adjacencyList.ContainsKey(vertex))
            _adjacencyList[vertex] = new List<T>();
    }

    // Kenar ekleme
    public void AddEdge(T from, T to)
    {
        AddVertex(from);
        AddVertex(to);

        _adjacencyList[from].Add(to);

        // Yönsüz graphta her iki yönde de kenar eklenir
        if (!_isDirected)
            _adjacencyList[to].Add(from);
    }

    // Düğümün komşularını getir
    public IReadOnlyList<T> GetNeighbors(T vertex)
    {
        return _adjacencyList.TryGetValue(vertex, out var neighbors)
            ? neighbors.AsReadOnly()
            : Array.Empty<T>();
    }

    // Tüm düğümleri getir
    public IEnumerable<T> GetVertices() => _adjacencyList.Keys;

    // Kenar var mı kontrolü
    public bool HasEdge(T from, T to)
    {
        return _adjacencyList.TryGetValue(from, out var neighbors)
               && neighbors.Contains(to);
    }

    // Düğüm sayısı
    public int VertexCount => _adjacencyList.Count;

    // Kenar sayısı
    public int EdgeCount
    {
        get
        {
            int total = _adjacencyList.Values.Sum(list => list.Count);
            return _isDirected ? total : total / 2;
        }
    }

    public override string ToString()
    {
        var sb = new System.Text.StringBuilder();
        foreach (var (vertex, neighbors) in _adjacencyList)
        {
            sb.Append($"{vertex} -> [{string.Join(", ", neighbors)}]");
            sb.AppendLine();
        }
        return sb.ToString();
    }
}

// Kullanım örneği
var graph = new Graph<string>(isDirected: false);
graph.AddEdge("A", "B");
graph.AddEdge("A", "C");
graph.AddEdge("B", "D");
graph.AddEdge("C", "D");
graph.AddEdge("D", "E");

Console.WriteLine(graph);
// A -> [B, C]
// B -> [A, D]
// C -> [A, D]
// D -> [B, C, E]
// E -> [D]
```

---

### 2. Soru

**BFS (Breadth-First Search) algoritmasını açıklayın ve C# ile implemente edin. Yönsüz bir graphta iki düğüm arasındaki en kısa yolu (edge sayısı olarak) bulun.**

**Cevap:**

BFS, bir graph üzerinde başlangıç düğümünden itibaren önce tüm komşuları, sonra komşuların komşularını ziyaret eden seviye seviye bir arama algoritmasıdır. **Queue (kuyruk)** veri yapısını kullanır. Ağırlıksız graphlarda kenar sayısı bakımından en kısa yolu garanti eder çünkü düğümleri uzaklık sırasına göre keşfeder.

**Zaman Karmaşıklığı:** O(V + E)
**Alan Karmaşıklığı:** O(V)

**Örnek Kod:**

```csharp
public class BfsAlgorithms
{
    // Temel BFS — tüm erişilebilir düğümleri ziyaret et
    public static List<T> BreadthFirstSearch<T>(
        Dictionary<T, List<T>> graph,
        T start) where T : notnull
    {
        var visited = new HashSet<T>();
        var queue = new Queue<T>();
        var result = new List<T>();

        visited.Add(start);
        queue.Enqueue(start);

        while (queue.Count > 0)
        {
            var current = queue.Dequeue();
            result.Add(current);

            foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
            {
                if (!visited.Contains(neighbor))
                {
                    visited.Add(neighbor);
                    queue.Enqueue(neighbor);
                }
            }
        }

        return result;
    }

    // En kısa yolu bul (unweighted graph)
    public static List<T>? ShortestPath<T>(
        Dictionary<T, List<T>> graph,
        T start,
        T end) where T : notnull
    {
        if (EqualityComparer<T>.Default.Equals(start, end))
            return new List<T> { start };

        var visited = new HashSet<T> { start };
        // Her düğüm için önceki düğümü sakla (path rekonstrüksiyonu için)
        var parent = new Dictionary<T, T>();
        var queue = new Queue<T>();
        queue.Enqueue(start);

        while (queue.Count > 0)
        {
            var current = queue.Dequeue();

            foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
            {
                if (visited.Contains(neighbor))
                    continue;

                visited.Add(neighbor);
                parent[neighbor] = current;

                if (EqualityComparer<T>.Default.Equals(neighbor, end))
                    return ReconstructPath(parent, start, end);

                queue.Enqueue(neighbor);
            }
        }

        return null; // Yol bulunamadı
    }

    // BFS ile seviye (level) bazlı dolaşma
    public static List<List<T>> BfsLevelOrder<T>(
        Dictionary<T, List<T>> graph,
        T start) where T : notnull
    {
        var levels = new List<List<T>>();
        var visited = new HashSet<T> { start };
        var queue = new Queue<T>();
        queue.Enqueue(start);

        while (queue.Count > 0)
        {
            int levelSize = queue.Count;
            var currentLevel = new List<T>();

            for (int i = 0; i < levelSize; i++)
            {
                var node = queue.Dequeue();
                currentLevel.Add(node);

                foreach (var neighbor in graph.GetValueOrDefault(node, new List<T>()))
                {
                    if (!visited.Contains(neighbor))
                    {
                        visited.Add(neighbor);
                        queue.Enqueue(neighbor);
                    }
                }
            }

            levels.Add(currentLevel);
        }

        return levels;
    }

    private static List<T> ReconstructPath<T>(
        Dictionary<T, T> parent,
        T start,
        T end) where T : notnull
    {
        var path = new List<T>();
        var current = end;

        while (!EqualityComparer<T>.Default.Equals(current, start))
        {
            path.Add(current);
            current = parent[current];
        }

        path.Add(start);
        path.Reverse();
        return path;
    }
}

// Kullanım
var graph = new Dictionary<string, List<string>>
{
    ["A"] = new List<string> { "B", "C" },
    ["B"] = new List<string> { "A", "D", "E" },
    ["C"] = new List<string> { "A", "F" },
    ["D"] = new List<string> { "B" },
    ["E"] = new List<string> { "B", "F" },
    ["F"] = new List<string> { "C", "E" }
};

var path = BfsAlgorithms.ShortestPath(graph, "A", "F");
Console.WriteLine(string.Join(" -> ", path!));
// A -> C -> F

var levels = BfsAlgorithms.BfsLevelOrder(graph, "A");
for (int i = 0; i < levels.Count; i++)
    Console.WriteLine($"Seviye {i}: [{string.Join(", ", levels[i])}]");
// Seviye 0: [A]
// Seviye 1: [B, C]
// Seviye 2: [D, E, F]
```

---

### 3. Soru

**DFS (Depth-First Search) algoritmasını hem recursive hem de iterative olarak implemente edin. Pre-order ve post-order dolaşma arasındaki fark nedir?**

**Cevap:**

DFS, bir pathı mümkün olduğunca derinlemesine takip eden, sonra geri dönerek (backtrack) diğer yolları deneyen bir arama algoritmasıdır. Recursive implementasyonda call stack kullanılır; iterative versiyonda ise explicit bir stack yapısı kullanılır.

- **Pre-order:** Düğüm önce işlenir, sonra komşulara geçilir (ziyaret sırası önemli olduğunda)
- **Post-order:** Önce tüm komşular işlenir, sonra düğüm işlenir (topological sort için kullanılır)

**Zaman Karmaşıklığı:** O(V + E)

**Örnek Kod:**

```csharp
public class DfsAlgorithms
{
    // Recursive DFS — pre-order
    public static List<T> DfsRecursive<T>(
        Dictionary<T, List<T>> graph,
        T start) where T : notnull
    {
        var visited = new HashSet<T>();
        var result = new List<T>();
        DfsHelper(graph, start, visited, result);
        return result;
    }

    private static void DfsHelper<T>(
        Dictionary<T, List<T>> graph,
        T current,
        HashSet<T> visited,
        List<T> result) where T : notnull
    {
        visited.Add(current);
        result.Add(current); // Pre-order: önce işle

        foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
        {
            if (!visited.Contains(neighbor))
                DfsHelper(graph, neighbor, visited, result);
        }
    }

    // Recursive DFS — post-order (Topological Sort için temel)
    public static List<T> DfsPostOrder<T>(
        Dictionary<T, List<T>> graph,
        T start) where T : notnull
    {
        var visited = new HashSet<T>();
        var result = new List<T>();
        DfsPostOrderHelper(graph, start, visited, result);
        return result;
    }

    private static void DfsPostOrderHelper<T>(
        Dictionary<T, List<T>> graph,
        T current,
        HashSet<T> visited,
        List<T> result) where T : notnull
    {
        visited.Add(current);

        foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
        {
            if (!visited.Contains(neighbor))
                DfsPostOrderHelper(graph, neighbor, visited, result);
        }

        result.Add(current); // Post-order: sonra işle
    }

    // Iterative DFS — explicit stack kullanımı
    public static List<T> DfsIterative<T>(
        Dictionary<T, List<T>> graph,
        T start) where T : notnull
    {
        var visited = new HashSet<T>();
        var stack = new Stack<T>();
        var result = new List<T>();

        stack.Push(start);

        while (stack.Count > 0)
        {
            var current = stack.Pop();

            if (visited.Contains(current))
                continue;

            visited.Add(current);
            result.Add(current);

            // Komşuları ters sırada ekle (sol-sağ sırasını korumak için)
            var neighbors = graph.GetValueOrDefault(current, new List<T>());
            for (int i = neighbors.Count - 1; i >= 0; i--)
            {
                if (!visited.Contains(neighbors[i]))
                    stack.Push(neighbors[i]);
            }
        }

        return result;
    }

    // Tüm düğümleri DFS ile dolaş (disconnected graph desteği)
    public static List<T> DfsAll<T>(
        Dictionary<T, List<T>> graph) where T : notnull
    {
        var visited = new HashSet<T>();
        var result = new List<T>();

        foreach (var vertex in graph.Keys)
        {
            if (!visited.Contains(vertex))
                DfsHelper(graph, vertex, visited, result);
        }

        return result;
    }
}

// Kullanım
var graph = new Dictionary<string, List<string>>
{
    ["A"] = new List<string> { "B", "C" },
    ["B"] = new List<string> { "D", "E" },
    ["C"] = new List<string> { "F" },
    ["D"] = new List<string>(),
    ["E"] = new List<string> { "F" },
    ["F"] = new List<string>()
};

var preOrder = DfsAlgorithms.DfsRecursive(graph, "A");
Console.WriteLine("Pre-order:  " + string.Join(", ", preOrder));
// Pre-order:  A, B, D, E, F, C

var postOrder = DfsAlgorithms.DfsPostOrder(graph, "A");
Console.WriteLine("Post-order: " + string.Join(", ", postOrder));
// Post-order: D, F, E, B, F, C, A  (F ve C sırası bağımlılığa göre)

var iterative = DfsAlgorithms.DfsIterative(graph, "A");
Console.WriteLine("Iterative:  " + string.Join(", ", iterative));
// Iterative:  A, B, D, E, F, C
```

---

### 4. Soru

**Yönsüz bir graphta döngü (cycle) tespiti nasıl yapılır? DFS kullanarak C# implementasyonu yapın.**

**Cevap:**

Yönsüz graphlarda döngü tespiti için DFS kullanılırken, bir düğümün zaten ziyaret edilmiş başka bir düğüme bağlandığını kontrol ederiz. Ancak dikkat edilmesi gereken nokta: geldiğimiz düğümü (parent) döngü olarak saymamamız gerekir, çünkü yönsüz graphta her kenar her iki yönde de mevcuttur.

**Örnek Kod:**

```csharp
public class CycleDetection
{
    // Yönsüz graphta döngü tespiti
    public static bool HasCycleUndirected<T>(
        Dictionary<T, List<T>> graph) where T : notnull
    {
        var visited = new HashSet<T>();

        // Her bileşen için ayrı ayrı kontrol et (disconnected graph)
        foreach (var vertex in graph.Keys)
        {
            if (!visited.Contains(vertex))
            {
                if (DfsDetectCycleUndirected(graph, vertex, default, visited))
                    return true;
            }
        }

        return false;
    }

    private static bool DfsDetectCycleUndirected<T>(
        Dictionary<T, List<T>> graph,
        T current,
        T? parent,
        HashSet<T> visited) where T : notnull
    {
        visited.Add(current);

        foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
        {
            if (!visited.Contains(neighbor))
            {
                if (DfsDetectCycleUndirected(graph, neighbor, current, visited))
                    return true;
            }
            else if (!EqualityComparer<T>.Default.Equals(neighbor, parent))
            {
                // Ziyaret edilmiş ama parent değil — döngü var!
                return true;
            }
        }

        return false;
    }

    // Yönlü graphta döngü tespiti — recursion stack takibi
    public static bool HasCycleDirected<T>(
        Dictionary<T, List<T>> graph) where T : notnull
    {
        var visited = new HashSet<T>();
        var recursionStack = new HashSet<T>(); // Mevcut DFS pathındaki düğümler

        foreach (var vertex in graph.Keys)
        {
            if (!visited.Contains(vertex))
            {
                if (DfsDetectCycleDirected(graph, vertex, visited, recursionStack))
                    return true;
            }
        }

        return false;
    }

    private static bool DfsDetectCycleDirected<T>(
        Dictionary<T, List<T>> graph,
        T current,
        HashSet<T> visited,
        HashSet<T> recursionStack) where T : notnull
    {
        visited.Add(current);
        recursionStack.Add(current);

        foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
        {
            if (!visited.Contains(neighbor))
            {
                if (DfsDetectCycleDirected(graph, neighbor, visited, recursionStack))
                    return true;
            }
            else if (recursionStack.Contains(neighbor))
            {
                // Mevcut DFS pathında — back edge — döngü var!
                return true;
            }
        }

        recursionStack.Remove(current); // Backtrack sırasında stack'ten çıkar
        return false;
    }
}

// Kullanım — yönsüz
var undirectedGraph = new Dictionary<string, List<string>>
{
    ["A"] = new List<string> { "B", "C" },
    ["B"] = new List<string> { "A", "D" },
    ["C"] = new List<string> { "A", "D" }, // A-B-D-C-A döngüsü oluşturur
    ["D"] = new List<string> { "B", "C" }
};

Console.WriteLine(CycleDetection.HasCycleUndirected(undirectedGraph)); // True

// Kullanım — yönlü
var directedGraph = new Dictionary<string, List<string>>
{
    ["A"] = new List<string> { "B" },
    ["B"] = new List<string> { "C" },
    ["C"] = new List<string> { "A" }, // A -> B -> C -> A döngüsü
    ["D"] = new List<string> { "C" }
};

Console.WriteLine(CycleDetection.HasCycleDirected(directedGraph)); // True
```

---

### 5. Soru

**Topological Sort (Topolojik Sıralama) nedir? DAG üzerinde Kahn's algoritması (BFS tabanlı) ile C# implementasyonu yapın. Build order sorununu çözün.**

**Cevap:**

Topological Sort, yönlü döngüsüz bir graphta (DAG) düğümleri, her düğümün bağımlı olduğu düğümlerden sonra gelecek şekilde sıralamaktır. Yazılım build sistemleri, görev zamanlama (task scheduling), ders önkoşul planlaması gibi bağımlılık problemlerinde kullanılır.

**Kahn's Algoritması:** Her düğümün in-degree (kendisine gelen kenar sayısı) takip edilir. In-degree 0 olan düğümler sıraya alınır, işlendikçe komşuların in-degree değeri azaltılır.

**Örnek Kod:**

```csharp
public class TopologicalSort
{
    // Kahn's Algoritması — BFS tabanlı
    public static List<T>? KahnTopologicalSort<T>(
        Dictionary<T, List<T>> graph) where T : notnull
    {
        // Her düğümün in-degree değerini hesapla
        var inDegree = new Dictionary<T, int>();

        foreach (var vertex in graph.Keys)
            inDegree.TryAdd(vertex, 0);

        foreach (var (_, neighbors) in graph)
        {
            foreach (var neighbor in neighbors)
            {
                inDegree[neighbor] = inDegree.GetValueOrDefault(neighbor, 0) + 1;
            }
        }

        // In-degree 0 olan tüm düğümleri kuyruğa ekle
        var queue = new Queue<T>();
        foreach (var (vertex, degree) in inDegree)
        {
            if (degree == 0)
                queue.Enqueue(vertex);
        }

        var result = new List<T>();

        while (queue.Count > 0)
        {
            var current = queue.Dequeue();
            result.Add(current);

            foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
            {
                inDegree[neighbor]--;

                if (inDegree[neighbor] == 0)
                    queue.Enqueue(neighbor);
            }
        }

        // Tüm düğümler işlenemediyse döngü var demektir
        return result.Count == graph.Count ? result : null;
    }

    // DFS tabanlı Topological Sort
    public static List<T>? DfsTopologicalSort<T>(
        Dictionary<T, List<T>> graph) where T : notnull
    {
        var visited = new HashSet<T>();
        var recursionStack = new HashSet<T>();
        var stack = new Stack<T>();

        foreach (var vertex in graph.Keys)
        {
            if (!visited.Contains(vertex))
            {
                if (!DfsHelper(graph, vertex, visited, recursionStack, stack))
                    return null; // Döngü tespit edildi
            }
        }

        return stack.ToList();
    }

    private static bool DfsHelper<T>(
        Dictionary<T, List<T>> graph,
        T current,
        HashSet<T> visited,
        HashSet<T> recursionStack,
        Stack<T> stack) where T : notnull
    {
        visited.Add(current);
        recursionStack.Add(current);

        foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
        {
            if (recursionStack.Contains(neighbor))
                return false; // Döngü var

            if (!visited.Contains(neighbor))
            {
                if (!DfsHelper(graph, neighbor, visited, recursionStack, stack))
                    return false;
            }
        }

        recursionStack.Remove(current);
        stack.Push(current); // Post-order: tüm bağımlılıklar işlendikten sonra ekle
        return true;
    }

    // Gerçek dünya örneği: Build order
    public static List<string>? FindBuildOrder(
        string[] projects,
        (string, string)[] dependencies)
    {
        var graph = new Dictionary<string, List<string>>();

        foreach (var project in projects)
            graph[project] = new List<string>();

        // (A, B) → A, B'ye bağımlı, yani B önce build edilmeli (B → A)
        foreach (var (dependent, dependency) in dependencies)
            graph[dependency].Add(dependent);

        return KahnTopologicalSort(graph);
    }
}

// Build order kullanımı
var projects = new[] { "a", "b", "c", "d", "e", "f" };
var dependencies = new (string, string)[]
{
    ("d", "a"), // d, a'ya bağımlı
    ("b", "f"), // b, f'ye bağımlı
    ("d", "b"), // d, b'ye bağımlı
    ("a", "f"), // a, f'ye bağımlı
    ("c", "d")  // c, d'ye bağımlı
};

var buildOrder = TopologicalSort.FindBuildOrder(projects, dependencies);
if (buildOrder != null)
    Console.WriteLine("Build sırası: " + string.Join(" -> ", buildOrder));
else
    Console.WriteLine("Döngüsel bağımlılık tespit edildi!");
// Build sırası: e -> f -> a -> b -> d -> c
```

---

### 6. Soru

**Dijkstra's Algoritması nedir ve ne zaman kullanılır? .NET 6+ ile gelen `PriorityQueue<TElement, TPriority>` kullanarak C# implementasyonu yapın.**

**Cevap:**

Dijkstra's algoritması, ağırlıklı bir graphta tek bir kaynaktan (single source) tüm diğer düğümlere olan en kısa yolu bulan greedy bir algoritmadır. Negatif ağırlıklı kenarlarla çalışmaz (bunun için Bellman-Ford kullanılır). .NET 6 ile gelen `PriorityQueue<TElement, TPriority>` min-heap implementasyonu bu algoritma için idealdir.

**Zaman Karmaşıklığı:** O((V + E) log V) — PriorityQueue ile

**Örnek Kod:**

```csharp
public class DijkstraAlgorithm
{
    public record Edge<T>(T To, int Weight);

    // Ağırlıklı graph için Dijkstra
    public static (Dictionary<T, int> Distances, Dictionary<T, T?> Previous)
        Dijkstra<T>(Dictionary<T, List<Edge<T>>> graph, T source)
        where T : notnull
    {
        var distances = new Dictionary<T, int>();
        var previous = new Dictionary<T, T?>();
        var visited = new HashSet<T>();

        // Tüm mesafeleri sonsuz olarak başlat
        foreach (var vertex in graph.Keys)
        {
            distances[vertex] = int.MaxValue;
            previous[vertex] = default;
        }

        distances[source] = 0;

        // .NET 6+ PriorityQueue — (düğüm, mesafe) çiftlerini mesafeye göre sıralar
        var priorityQueue = new PriorityQueue<T, int>();
        priorityQueue.Enqueue(source, 0);

        while (priorityQueue.Count > 0)
        {
            priorityQueue.TryDequeue(out var current, out var currentDist);

            // Zaten daha kısa bir yol bulunduysa atla (lazy deletion)
            if (visited.Contains(current))
                continue;

            visited.Add(current);

            if (!graph.TryGetValue(current, out var neighbors))
                continue;

            foreach (var edge in neighbors)
            {
                if (visited.Contains(edge.To))
                    continue;

                int newDist = distances[current] == int.MaxValue
                    ? int.MaxValue
                    : distances[current] + edge.Weight;

                if (newDist < distances.GetValueOrDefault(edge.To, int.MaxValue))
                {
                    distances[edge.To] = newDist;
                    previous[edge.To] = current;
                    priorityQueue.Enqueue(edge.To, newDist);
                }
            }
        }

        return (distances, previous);
    }

    // Belirli bir hedefe giden yolu rekonstrükte et
    public static List<T> GetPath<T>(
        Dictionary<T, T?> previous,
        T source,
        T target) where T : notnull
    {
        var path = new List<T>();
        var current = target;

        while (!EqualityComparer<T>.Default.Equals(current, source))
        {
            if (previous[current] == null)
                return new List<T>(); // Yol yok

            path.Add(current);
            current = previous[current]!;
        }

        path.Add(source);
        path.Reverse();
        return path;
    }
}

// Kullanım — harita/navigasyon senaryosu
var cityGraph = new Dictionary<string, List<DijkstraAlgorithm.Edge<string>>>
{
    ["Istanbul"] = new()
    {
        new("Ankara", 450),
        new("Bursa", 150)
    },
    ["Ankara"] = new()
    {
        new("Istanbul", 450),
        new("Konya", 260),
        new("Izmir", 590)
    },
    ["Bursa"] = new()
    {
        new("Istanbul", 150),
        new("Izmir", 330)
    },
    ["Izmir"] = new()
    {
        new("Bursa", 330),
        new("Ankara", 590),
        new("Konya", 350)
    },
    ["Konya"] = new()
    {
        new("Ankara", 260),
        new("Izmir", 350)
    }
};

var (distances, previous) = DijkstraAlgorithm.Dijkstra(cityGraph, "Istanbul");

Console.WriteLine("Istanbul'dan mesafeler:");
foreach (var (city, dist) in distances.OrderBy(x => x.Value))
    Console.WriteLine($"  {city}: {dist} km");

var path = DijkstraAlgorithm.GetPath(previous, "Istanbul", "Konya");
Console.WriteLine("\nIstanbul -> Konya en kısa yol: " + string.Join(" -> ", path));
Console.WriteLine($"Toplam mesafe: {distances["Konya"]} km");
// Istanbul -> Ankara -> Konya
// Toplam mesafe: 710 km
```

---

### 7. Soru

**Connected Components (Bağlı Bileşenler) nedir? Yönsüz bir graphta kaç ayrı bağlı bileşen olduğunu ve her bileşenin düğümlerini bulan algoritmayı yazın.**

**Cevap:**

Bağlı bileşen (connected component), bir graphta birbirinden erişilebilir düğümler grubudur. Yönsüz graphlarda iki düğüm aynı bileşendeyse birbirlerine bir path üzerinden ulaşılabilir. Bu problem, sosyal ağ analizinde (birbirini tanıyan kişi grupları), harita üzerinde izole bölge tespitinde kullanılır.

**Örnek Kod:**

```csharp
public class ConnectedComponents
{
    // DFS ile bağlı bileşenleri bul
    public static List<List<T>> FindConnectedComponents<T>(
        Dictionary<T, List<T>> graph) where T : notnull
    {
        var visited = new HashSet<T>();
        var components = new List<List<T>>();

        foreach (var vertex in graph.Keys)
        {
            if (!visited.Contains(vertex))
            {
                var component = new List<T>();
                DfsComponent(graph, vertex, visited, component);
                components.Add(component);
            }
        }

        return components;
    }

    private static void DfsComponent<T>(
        Dictionary<T, List<T>> graph,
        T current,
        HashSet<T> visited,
        List<T> component) where T : notnull
    {
        visited.Add(current);
        component.Add(current);

        foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
        {
            if (!visited.Contains(neighbor))
                DfsComponent(graph, neighbor, visited, component);
        }
    }

    // Union-Find (Disjoint Set Union) ile bağlı bileşen sayısı
    public static int CountComponentsUnionFind(int n, int[][] edges)
    {
        var parent = Enumerable.Range(0, n).ToArray();
        var rank = new int[n];

        int Find(int x)
        {
            if (parent[x] != x)
                parent[x] = Find(parent[x]); // Path compression
            return parent[x];
        }

        bool Union(int x, int y)
        {
            int px = Find(x), py = Find(y);
            if (px == py) return false;

            // Union by rank
            if (rank[px] < rank[py]) (px, py) = (py, px);
            parent[py] = px;
            if (rank[px] == rank[py]) rank[px]++;
            return true;
        }

        int components = n;
        foreach (var edge in edges)
        {
            if (Union(edge[0], edge[1]))
                components--;
        }

        return components;
    }

    // Strongly Connected Components için Kosaraju algoritması (yönlü graph)
    public static List<List<T>> KosarajuSCC<T>(
        Dictionary<T, List<T>> graph) where T : notnull
    {
        // 1. Adım: Finish time sırasına göre düğümleri stack'e ekle
        var visited = new HashSet<T>();
        var finishStack = new Stack<T>();

        foreach (var vertex in graph.Keys)
        {
            if (!visited.Contains(vertex))
                Dfs1(graph, vertex, visited, finishStack);
        }

        // 2. Adım: Graphı ters çevir
        var transposed = TransposeGraph(graph);

        // 3. Adım: Ters graph üzerinde finish time sırasına göre DFS
        visited.Clear();
        var sccs = new List<List<T>>();

        while (finishStack.Count > 0)
        {
            var vertex = finishStack.Pop();
            if (!visited.Contains(vertex))
            {
                var scc = new List<T>();
                Dfs2(transposed, vertex, visited, scc);
                sccs.Add(scc);
            }
        }

        return sccs;
    }

    private static void Dfs1<T>(
        Dictionary<T, List<T>> graph, T current,
        HashSet<T> visited, Stack<T> finishStack) where T : notnull
    {
        visited.Add(current);
        foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
            if (!visited.Contains(neighbor))
                Dfs1(graph, neighbor, visited, finishStack);
        finishStack.Push(current);
    }

    private static void Dfs2<T>(
        Dictionary<T, List<T>> graph, T current,
        HashSet<T> visited, List<T> scc) where T : notnull
    {
        visited.Add(current);
        scc.Add(current);
        foreach (var neighbor in graph.GetValueOrDefault(current, new List<T>()))
            if (!visited.Contains(neighbor))
                Dfs2(graph, neighbor, visited, scc);
    }

    private static Dictionary<T, List<T>> TransposeGraph<T>(
        Dictionary<T, List<T>> graph) where T : notnull
    {
        var transposed = new Dictionary<T, List<T>>();
        foreach (var vertex in graph.Keys)
            transposed[vertex] = new List<T>();

        foreach (var (vertex, neighbors) in graph)
            foreach (var neighbor in neighbors)
                transposed[neighbor].Add(vertex);

        return transposed;
    }
}

// Kullanım
var graph = new Dictionary<string, List<string>>
{
    ["A"] = new List<string> { "B" },
    ["B"] = new List<string> { "A" },
    ["C"] = new List<string> { "D" },
    ["D"] = new List<string> { "C" },
    ["E"] = new List<string>() // Yalnız düğüm
};

var components = ConnectedComponents.FindConnectedComponents(graph);
Console.WriteLine($"Bağlı bileşen sayısı: {components.Count}");
for (int i = 0; i < components.Count; i++)
    Console.WriteLine($"  Bileşen {i + 1}: [{string.Join(", ", components[i])}]");
// Bağlı bileşen sayısı: 3
// Bileşen 1: [A, B]
// Bileşen 2: [C, D]
// Bileşen 3: [E]

// Union-Find kullanımı
int nodeCount = 5;
int[][] edges = { new[] { 0, 1 }, new[] { 1, 2 }, new[] { 3, 4 } };
Console.WriteLine($"Bileşen sayısı (Union-Find): {ConnectedComponents.CountComponentsUnionFind(nodeCount, edges)}");
// Bileşen sayısı (Union-Find): 2
```

---

### 8. Soru

**Gerçek bir mülakat sorusu: "Number of Islands" — 2D bir grid üzerinde '1' (kara) ve '0' (su) karakterlerinden oluşan bir matrixte ada sayısını bulun. BFS ve DFS çözümlerini karşılaştırın.**

**Cevap:**

Bu LeetCode tarzı klasik bir graph sorusudur. Grid'i bir graph olarak düşünebiliriz; her '1' hücresi bir düğüm, komşu '1' hücreleri arasındaki bağlantılar ise kenardır. Her DFS/BFS çağrısı bir adayı tamamen işaretler. Bu teknik "flood fill" olarak da bilinir.

**Zaman Karmaşıklığı:** O(M × N) — M satır, N sütun
**Alan Karmaşıklığı:** O(M × N) — worst case (tüm grid kara)

**Örnek Kod:**

```csharp
public class NumberOfIslands
{
    private static readonly int[] DeltaRow = { -1, 1, 0, 0 };
    private static readonly int[] DeltaCol = { 0, 0, -1, 1 };

    // DFS çözümü — recursive flood fill
    public static int CountIslandsDfs(char[][] grid)
    {
        if (grid == null || grid.Length == 0)
            return 0;

        int rows = grid.Length;
        int cols = grid[0].Length;
        int count = 0;

        for (int r = 0; r < rows; r++)
        {
            for (int c = 0; c < cols; c++)
            {
                if (grid[r][c] == '1')
                {
                    count++;
                    DfsFloodFill(grid, r, c, rows, cols);
                }
            }
        }

        return count;
    }

    private static void DfsFloodFill(char[][] grid, int row, int col, int rows, int cols)
    {
        if (row < 0 || row >= rows || col < 0 || col >= cols || grid[row][col] != '1')
            return;

        grid[row][col] = '0'; // Ziyaret edildi olarak işaretle (in-place)

        for (int d = 0; d < 4; d++)
            DfsFloodFill(grid, row + DeltaRow[d], col + DeltaCol[d], rows, cols);
    }

    // BFS çözümü — queue ile genişleme
    public static int CountIslandsBfs(char[][] grid)
    {
        if (grid == null || grid.Length == 0)
            return 0;

        int rows = grid.Length;
        int cols = grid[0].Length;
        int count = 0;

        for (int r = 0; r < rows; r++)
        {
            for (int c = 0; c < cols; c++)
            {
                if (grid[r][c] == '1')
                {
                    count++;
                    BfsFloodFill(grid, r, c, rows, cols);
                }
            }
        }

        return count;
    }

    private static void BfsFloodFill(char[][] grid, int startRow, int startCol, int rows, int cols)
    {
        var queue = new Queue<(int Row, int Col)>();
        grid[startRow][startCol] = '0';
        queue.Enqueue((startRow, startCol));

        while (queue.Count > 0)
        {
            var (row, col) = queue.Dequeue();

            for (int d = 0; d < 4; d++)
            {
                int newRow = row + DeltaRow[d];
                int newCol = col + DeltaCol[d];

                if (newRow >= 0 && newRow < rows &&
                    newCol >= 0 && newCol < cols &&
                    grid[newRow][newCol] == '1')
                {
                    grid[newRow][newCol] = '0';
                    queue.Enqueue((newRow, newCol));
                }
            }
        }
    }

    // Orijinal grid'i koruyarak çözüm (visited matrix ile)
    public static int CountIslandsNonDestructive(char[][] grid)
    {
        if (grid == null || grid.Length == 0)
            return 0;

        int rows = grid.Length;
        int cols = grid[0].Length;
        bool[,] visited = new bool[rows, cols];
        int count = 0;

        for (int r = 0; r < rows; r++)
        {
            for (int c = 0; c < cols; c++)
            {
                if (grid[r][c] == '1' && !visited[r, c])
                {
                    count++;
                    DfsWithVisited(grid, r, c, rows, cols, visited);
                }
            }
        }

        return count;
    }

    private static void DfsWithVisited(
        char[][] grid, int row, int col,
        int rows, int cols, bool[,] visited)
    {
        if (row < 0 || row >= rows || col < 0 || col >= cols ||
            visited[row, col] || grid[row][col] != '1')
            return;

        visited[row, col] = true;

        for (int d = 0; d < 4; d++)
            DfsWithVisited(grid, row + DeltaRow[d], col + DeltaCol[d], rows, cols, visited);
    }
}

// Test
char[][] grid1 =
{
    new[] { '1', '1', '1', '1', '0' },
    new[] { '1', '1', '0', '1', '0' },
    new[] { '1', '1', '0', '0', '0' },
    new[] { '0', '0', '0', '0', '0' }
};

// grid'i kopyala (DFS grid'i değiştirir)
char[][] grid2 =
{
    new[] { '1', '1', '0', '0', '0' },
    new[] { '1', '1', '0', '0', '0' },
    new[] { '0', '0', '1', '0', '0' },
    new[] { '0', '0', '0', '1', '1' }
};

Console.WriteLine(NumberOfIslands.CountIslandsDfs(grid1));  // 1
Console.WriteLine(NumberOfIslands.CountIslandsBfs(grid2));  // 3

// DFS vs BFS Karşılaştırma:
// DFS: Call stack kullanır, büyük grids'de StackOverflowException riski
// BFS: Explicit queue kullanır, büyük girişlerde daha güvenli
// Her ikisi de O(M*N) zaman ve alan karmaşıklığına sahiptir
```

---

## Özet Karşılaştırma Tablosu

| Algoritma | Zaman | Alan | Kullanım Alanı |
|---|---|---|---|
| BFS | O(V+E) | O(V) | En kısa yol (unweighted), level-order |
| DFS | O(V+E) | O(V) | Cycle detection, pathfinding, topo sort |
| Dijkstra | O((V+E) log V) | O(V) | En kısa yol (weighted, pozitif) |
| Topological Sort (Kahn) | O(V+E) | O(V) | Build order, task scheduling |
| Connected Components | O(V+E) | O(V) | Cluster analizi, izole bölge tespiti |
| Union-Find | O(α(N)) amortized | O(N) | Dynamic connectivity, Kruskal's MST |

**Mülakatlarda Dikkat Edilecek Noktalar:**

- Ziyaret edilmiş düğümleri (`HashSet`) takip etmeyi unutmayın, aksi takdirde sonsuz döngüye girersiniz.
- Yönsüz graphlarda cycle detection için parent düğümü saklayın.
- Yönlü graphlarda cycle detection için recursion stack ayrı tutulmalıdır.
- Dijkstra negatif kenarlarla çalışmaz; Bellman-Ford kullanın.
- `PriorityQueue<TElement, TPriority>` .NET 6 ile geldi; önceki versiyonlarda SortedSet veya harici kütüphane gerekir.
- Graph disconnected olabilir; tüm düğümleri dolaşmak için her düğümden DFS/BFS başlatın.
