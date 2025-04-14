# Array Algorithms

## Genel Bakış

Dizi algoritmaları, yazılım geliştirmede en sık kullanılan algoritma türlerinden biridir. Bu bölümde, diziler üzerinde yapılan temel işlemleri ve bunların C# implementasyonlarını inceleyeceğiz.

## Temel Dizi İşlemleri

### 1. En Büyük ve En Küçük Eleman Bulma

```csharp
public class MinMaxFinder
{
    public (int min, int max) FindMinMax(int[] array)
    {
        if (array == null || array.Length == 0)
            throw new ArgumentException("Dizi boş olamaz");

        int min = array[0];
        int max = array[0];

        for (int i = 1; i < array.Length; i++)
        {
            if (array[i] < min)
                min = array[i];
            if (array[i] > max)
                max = array[i];
        }

        return (min, max);
    }
}
```

### 2. Diziyi Tersine Çevirme

```csharp
public class ArrayReverser
{
    public void ReverseArray(int[] array)
    {
        if (array == null)
            throw new ArgumentNullException(nameof(array));

        int start = 0;
        int end = array.Length - 1;

        while (start < end)
        {
            int temp = array[start];
            array[start] = array[end];
            array[end] = temp;
            start++;
            end--;
        }
    }
}
```

### 3. Eksik Sayıyı Bulma

```csharp
public class MissingNumberFinder
{
    public int FindMissingNumber(int[] array)
    {
        if (array == null)
            throw new ArgumentNullException(nameof(array));

        int n = array.Length + 1;
        int expectedSum = n * (n + 1) / 2;
        int actualSum = array.Sum();

        return expectedSum - actualSum;
    }
}
```

## Arama Algoritmaları

### 1. Linear Search

```csharp
public class LinearSearch
{
    public int Search(int[] array, int target)
    {
        for (int i = 0; i < array.Length; i++)
        {
            if (array[i] == target)
                return i;
        }
        return -1;
    }
}
```

### 2. Binary Search

```csharp
public class BinarySearch
{
    public int Search(int[] array, int target)
    {
        int left = 0;
        int right = array.Length - 1;

        while (left <= right)
        {
            int mid = left + (right - left) / 2;

            if (array[mid] == target)
                return mid;

            if (array[mid] < target)
                left = mid + 1;
            else
                right = mid - 1;
        }

        return -1;
    }
}
```

## Performans Analizi

| Algoritma | En İyi Durum | Ortalama Durum | En Kötü Durum | Bellek Kullanımı |
|-----------|-------------|----------------|---------------|------------------|
| Linear Search | O(1) | O(n) | O(n) | O(1) |
| Binary Search | O(1) | O(log n) | O(log n) | O(1) |
| Min/Max Bulma | O(n) | O(n) | O(n) | O(1) |
| Tersine Çevirme | O(n) | O(n) | O(n) | O(1) |

## Best Practices

1. Dizi sınırlarını kontrol et
2. Null kontrolü yap
3. Gereksiz döngülerden kaçın
4. Bellek kullanımını optimize et
5. Hata durumlarını yönet

## Örnek Uygulamalar

1. Dizi sıralama
2. Dizi filtreleme
3. Dizi birleştirme
4. Dizi karşılaştırma
5. Dizi dönüştürme

## Kaynaklar

- [Array Class Documentation](https://docs.microsoft.com/en-us/dotnet/api/system.array)
- [C# Arrays Tutorial](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/arrays/)
- [Array Algorithms in C#](https://www.geeksforgeeks.org/array-data-structure/) 