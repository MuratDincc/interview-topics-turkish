# Linked List Algorithms

## Genel Bakış

Bağlı liste algoritmaları, veri yapılarında sıralı veri depolamak için kullanılan temel algoritmalardır. Bu bölümde, bağlı liste işlemleri için kullanılan temel algoritmaları ve bunların C# implementasyonlarını inceleyeceğiz.

## Temel Bağlı Liste İşlemleri

### 1. Bağlı Liste Düğümü

```csharp
public class ListNode
{
    public int Value { get; set; }
    public ListNode Next { get; set; }

    public ListNode(int value)
    {
        Value = value;
        Next = null;
    }
}
```

### 2. Bağlı Liste Tersine Çevirme

```csharp
public class LinkedListReverser
{
    public ListNode ReverseList(ListNode head)
    {
        ListNode prev = null;
        ListNode current = head;
        ListNode next = null;

        while (current != null)
        {
            next = current.Next;
            current.Next = prev;
            prev = current;
            current = next;
        }

        return prev;
    }
}
```

### 3. Bağlı Liste Birleştirme

```csharp
public class LinkedListMerger
{
    public ListNode MergeTwoLists(ListNode l1, ListNode l2)
    {
        ListNode dummy = new ListNode(0);
        ListNode current = dummy;

        while (l1 != null && l2 != null)
        {
            if (l1.Value <= l2.Value)
            {
                current.Next = l1;
                l1 = l1.Next;
            }
            else
            {
                current.Next = l2;
                l2 = l2.Next;
            }
            current = current.Next;
        }

        current.Next = l1 ?? l2;
        return dummy.Next;
    }
}
```

## İleri Bağlı Liste Algoritmaları

### 1. Döngü Tespiti

```csharp
public class CycleDetector
{
    public bool HasCycle(ListNode head)
    {
        if (head == null || head.Next == null)
            return false;

        ListNode slow = head;
        ListNode fast = head.Next;

        while (slow != fast)
        {
            if (fast == null || fast.Next == null)
                return false;

            slow = slow.Next;
            fast = fast.Next.Next;
        }

        return true;
    }
}
```

### 2. Ortadaki Düğümü Bulma

```csharp
public class MiddleNodeFinder
{
    public ListNode FindMiddle(ListNode head)
    {
        ListNode slow = head;
        ListNode fast = head;

        while (fast != null && fast.Next != null)
        {
            slow = slow.Next;
            fast = fast.Next.Next;
        }

        return slow;
    }
}
```

### 3. K'ıncı Düğümü Silme

```csharp
public class KthNodeRemover
{
    public ListNode RemoveNthFromEnd(ListNode head, int n)
    {
        ListNode dummy = new ListNode(0);
        dummy.Next = head;
        ListNode first = dummy;
        ListNode second = dummy;

        for (int i = 1; i <= n + 1; i++)
        {
            first = first.Next;
        }

        while (first != null)
        {
            first = first.Next;
            second = second.Next;
        }

        second.Next = second.Next.Next;
        return dummy.Next;
    }
}
```

## Performans Analizi

| Algoritma | En İyi Durum | Ortalama Durum | En Kötü Durum | Bellek Kullanımı |
|-----------|-------------|----------------|---------------|------------------|
| Tersine Çevirme | O(n) | O(n) | O(n) | O(1) |
| Birleştirme | O(n) | O(n) | O(n) | O(1) |
| Döngü Tespiti | O(n) | O(n) | O(n) | O(1) |
| Ortadaki Düğüm | O(n) | O(n) | O(n) | O(1) |

## Best Practices

1. Null kontrollerini yap
2. Bellek sızıntılarını önle
3. Döngüsel referansları kontrol et
4. Performansı optimize et
5. Hata durumlarını yönet

## Örnek Uygulamalar

1. Veri yapısı implementasyonu
2. Algoritma optimizasyonu
3. Bellek yönetimi
4. Veri işleme
5. Sistem tasarımı

## Kaynaklar

- [Linked List in C#](https://www.geeksforgeeks.org/linked-list-implementation-in-c-sharp/)
- [C# Collections](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic)
- [Data Structures and Algorithms](https://www.tutorialspoint.com/data_structures_algorithms/index.htm) 