# Hash Table Algorithms

## Genel Bakış

Hash Table algoritmaları, veri yapılarında hızlı erişim ve depolama için kullanılan temel algoritmalardır. Bu bölümde, hash tablosu işlemleri için kullanılan temel algoritmaları ve bunların C# implementasyonlarını inceleyeceğiz.

## Temel Hash Table İşlemleri

### 1. Hash Fonksiyonu

```csharp
public class HashFunction
{
    public int GetHash(string key, int tableSize)
    {
        int hash = 0;
        foreach (char c in key)
        {
            hash = (hash * 31 + c) % tableSize;
        }
        return hash;
    }
}
```

### 2. Hash Table Oluşturma

```csharp
public class HashTable<TKey, TValue>
{
    private class HashNode
    {
        public TKey Key { get; set; }
        public TValue Value { get; set; }
        public HashNode Next { get; set; }

        public HashNode(TKey key, TValue value)
        {
            Key = key;
            Value = value;
            Next = null;
        }
    }

    private HashNode[] buckets;
    private int size;
    private readonly int capacity;

    public HashTable(int capacity = 16)
    {
        this.capacity = capacity;
        buckets = new HashNode[capacity];
        size = 0;
    }
}
```

### 3. Ekleme ve Arama

```csharp
public class HashTable<TKey, TValue>
{
    // ... önceki kod ...

    public void Add(TKey key, TValue value)
    {
        int index = GetBucketIndex(key);
        HashNode head = buckets[index];

        while (head != null)
        {
            if (head.Key.Equals(key))
            {
                head.Value = value;
                return;
            }
            head = head.Next;
        }

        HashNode newNode = new HashNode(key, value);
        newNode.Next = buckets[index];
        buckets[index] = newNode;
        size++;
    }

    public TValue Get(TKey key)
    {
        int index = GetBucketIndex(key);
        HashNode head = buckets[index];

        while (head != null)
        {
            if (head.Key.Equals(key))
                return head.Value;
            head = head.Next;
        }

        throw new KeyNotFoundException();
    }

    private int GetBucketIndex(TKey key)
    {
        return Math.Abs(key.GetHashCode() % capacity);
    }
}
```

## İleri Hash Table Algoritmaları

### 1. Çakışma Çözümleme (Chaining)

```csharp
public class HashTableWithChaining<TKey, TValue>
{
    private List<HashNode>[] buckets;
    private int size;
    private readonly int capacity;

    public HashTableWithChaining(int capacity = 16)
    {
        this.capacity = capacity;
        buckets = new List<HashNode>[capacity];
        for (int i = 0; i < capacity; i++)
        {
            buckets[i] = new List<HashNode>();
        }
        size = 0;
    }

    public void Add(TKey key, TValue value)
    {
        int index = GetBucketIndex(key);
        var bucket = buckets[index];

        foreach (var node in bucket)
        {
            if (node.Key.Equals(key))
            {
                node.Value = value;
                return;
            }
        }

        bucket.Add(new HashNode(key, value));
        size++;
    }
}
```

### 2. Açık Adresleme (Open Addressing)

```csharp
public class HashTableWithOpenAddressing<TKey, TValue>
{
    private class HashEntry
    {
        public TKey Key { get; set; }
        public TValue Value { get; set; }
        public bool IsDeleted { get; set; }

        public HashEntry(TKey key, TValue value)
        {
            Key = key;
            Value = value;
            IsDeleted = false;
        }
    }

    private HashEntry[] table;
    private int size;
    private readonly int capacity;

    public HashTableWithOpenAddressing(int capacity = 16)
    {
        this.capacity = capacity;
        table = new HashEntry[capacity];
        size = 0;
    }

    public void Add(TKey key, TValue value)
    {
        int index = GetBucketIndex(key);
        int startIndex = index;

        do
        {
            if (table[index] == null || table[index].IsDeleted)
            {
                table[index] = new HashEntry(key, value);
                size++;
                return;
            }

            if (table[index].Key.Equals(key))
            {
                table[index].Value = value;
                return;
            }

            index = (index + 1) % capacity;
        } while (index != startIndex);

        throw new InvalidOperationException("Hash table is full");
    }
}
```

### 3. Dinamik Yeniden Boyutlandırma

```csharp
public class HashTable<TKey, TValue>
{
    // ... önceki kod ...

    private void Resize()
    {
        int newCapacity = capacity * 2;
        HashNode[] newBuckets = new HashNode[newCapacity];

        foreach (var bucket in buckets)
        {
            HashNode current = bucket;
            while (current != null)
            {
                int newIndex = Math.Abs(current.Key.GetHashCode() % newCapacity);
                HashNode next = current.Next;
                current.Next = newBuckets[newIndex];
                newBuckets[newIndex] = current;
                current = next;
            }
        }

        buckets = newBuckets;
        capacity = newCapacity;
    }

    public void Add(TKey key, TValue value)
    {
        if (size >= capacity * 0.75)
        {
            Resize();
        }
        // ... mevcut ekleme kodu ...
    }
}
```

## Performans Analizi

| Algoritma | En İyi Durum | Ortalama Durum | En Kötü Durum | Bellek Kullanımı |
|-----------|-------------|----------------|---------------|------------------|
| Ekleme | O(1) | O(1) | O(n) | O(n) |
| Arama | O(1) | O(1) | O(n) | O(n) |
| Silme | O(1) | O(1) | O(n) | O(n) |
| Yeniden Boyutlandırma | O(n) | O(n) | O(n) | O(n) |

## Best Practices

1. İyi bir hash fonksiyonu seç
2. Çakışma çözümleme stratejisini dikkatli seç
3. Yük faktörünü kontrol et
4. Bellek kullanımını optimize et
5. Hata durumlarını yönet

## Örnek Uygulamalar

1. Sözlük uygulamaları
2. Önbellek sistemleri
3. Veri indeksleme
4. Veri filtreleme
5. Sistem tasarımı

## Kaynaklar

- [Hash Table in C#](https://www.geeksforgeeks.org/implementing-hash-table-in-c-sharp/)
- [C# Dictionary](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2)
- [Data Structures and Algorithms](https://www.tutorialspoint.com/data_structures_algorithms/index.htm) 