# Sorting Algorithms

## Genel Bakış

Sıralama algoritmaları, veri yapılarını belirli bir düzene göre sıralamak için kullanılan temel algoritmalardır. Bu bölümde, yaygın sıralama algoritmalarını ve bunların C# implementasyonlarını inceleyeceğiz.

## Temel Sıralama Algoritmaları

### 1. Bubble Sort

```csharp
public class BubbleSort
{
    public void Sort(int[] array)
    {
        int n = array.Length;
        for (int i = 0; i < n - 1; i++)
        {
            for (int j = 0; j < n - i - 1; j++)
            {
                if (array[j] > array[j + 1])
                {
                    int temp = array[j];
                    array[j] = array[j + 1];
                    array[j + 1] = temp;
                }
            }
        }
    }
}
```

### 2. Quick Sort

```csharp
public class QuickSort
{
    public void Sort(int[] array)
    {
        Sort(array, 0, array.Length - 1);
    }

    private void Sort(int[] array, int low, int high)
    {
        if (low < high)
        {
            int pi = Partition(array, low, high);
            Sort(array, low, pi - 1);
            Sort(array, pi + 1, high);
        }
    }

    private int Partition(int[] array, int low, int high)
    {
        int pivot = array[high];
        int i = low - 1;

        for (int j = low; j < high; j++)
        {
            if (array[j] < pivot)
            {
                i++;
                int temp = array[i];
                array[i] = array[j];
                array[j] = temp;
            }
        }

        int temp1 = array[i + 1];
        array[i + 1] = array[high];
        array[high] = temp1;

        return i + 1;
    }
}
```

### 3. Merge Sort

```csharp
public class MergeSort
{
    public void Sort(int[] array)
    {
        Sort(array, 0, array.Length - 1);
    }

    private void Sort(int[] array, int left, int right)
    {
        if (left < right)
        {
            int mid = left + (right - left) / 2;
            Sort(array, left, mid);
            Sort(array, mid + 1, right);
            Merge(array, left, mid, right);
        }
    }

    private void Merge(int[] array, int left, int mid, int right)
    {
        int n1 = mid - left + 1;
        int n2 = right - mid;

        int[] L = new int[n1];
        int[] R = new int[n2];

        Array.Copy(array, left, L, 0, n1);
        Array.Copy(array, mid + 1, R, 0, n2);

        int i = 0, j = 0;
        int k = left;

        while (i < n1 && j < n2)
        {
            if (L[i] <= R[j])
            {
                array[k] = L[i];
                i++;
            }
            else
            {
                array[k] = R[j];
                j++;
            }
            k++;
        }

        while (i < n1)
        {
            array[k] = L[i];
            i++;
            k++;
        }

        while (j < n2)
        {
            array[k] = R[j];
            j++;
            k++;
        }
    }
}
```

## İleri Sıralama Algoritmaları

### 1. Heap Sort

```csharp
public class HeapSort
{
    public void Sort(int[] array)
    {
        int n = array.Length;

        for (int i = n / 2 - 1; i >= 0; i--)
            Heapify(array, n, i);

        for (int i = n - 1; i > 0; i--)
        {
            int temp = array[0];
            array[0] = array[i];
            array[i] = temp;

            Heapify(array, i, 0);
        }
    }

    private void Heapify(int[] array, int n, int i)
    {
        int largest = i;
        int left = 2 * i + 1;
        int right = 2 * i + 2;

        if (left < n && array[left] > array[largest])
            largest = left;

        if (right < n && array[right] > array[largest])
            largest = right;

        if (largest != i)
        {
            int swap = array[i];
            array[i] = array[largest];
            array[largest] = swap;

            Heapify(array, n, largest);
        }
    }
}
```

### 2. Counting Sort

```csharp
public class CountingSort
{
    public void Sort(int[] array)
    {
        int max = array.Max();
        int min = array.Min();
        int range = max - min + 1;

        int[] count = new int[range];
        int[] output = new int[array.Length];

        for (int i = 0; i < array.Length; i++)
            count[array[i] - min]++;

        for (int i = 1; i < count.Length; i++)
            count[i] += count[i - 1];

        for (int i = array.Length - 1; i >= 0; i--)
        {
            output[count[array[i] - min] - 1] = array[i];
            count[array[i] - min]--;
        }

        Array.Copy(output, array, array.Length);
    }
}
```

## Performans Analizi

| Algoritma | En İyi Durum | Ortalama Durum | En Kötü Durum | Bellek Kullanımı |
|-----------|-------------|----------------|---------------|------------------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) |
| Counting Sort | O(n + k) | O(n + k) | O(n + k) | O(n + k) |

## Best Practices

1. Veri boyutunu dikkate al
2. Veri tipini göz önünde bulundur
3. Bellek kullanımını optimize et
4. Kararlılık gereksinimlerini değerlendir
5. Hata durumlarını yönet

## Örnek Uygulamalar

1. Büyük veri setleri
2. Kısmi sıralı veriler
3. Tekrar eden elemanlar
4. Karma veri tipleri
5. Özel sıralama kriterleri

## Kaynaklar

- [Sorting Algorithms in C#](https://www.geeksforgeeks.org/sorting-algorithms/)
- [C# Array.Sort](https://docs.microsoft.com/en-us/dotnet/api/system.array.sort)
- [Sorting Algorithm Comparison](https://www.toptal.com/developers/sorting-algorithms) 