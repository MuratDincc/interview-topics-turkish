# Tree Algorithms

## Genel Bakış

Ağaç algoritmaları, hiyerarşik veri yapılarını işlemek için kullanılan temel algoritmalardır. Bu bölümde, ağaç veri yapıları ve bunların C# implementasyonlarını inceleyeceğiz.

## Temel Ağaç İşlemleri

### 1. Ağaç Düğümü

```csharp
public class TreeNode
{
    public int Value { get; set; }
    public TreeNode Left { get; set; }
    public TreeNode Right { get; set; }

    public TreeNode(int value)
    {
        Value = value;
        Left = null;
        Right = null;
    }
}
```

### 2. Ağaç Yüksekliği

```csharp
public class TreeHeightCalculator
{
    public int CalculateHeight(TreeNode root)
    {
        if (root == null)
            return 0;

        int leftHeight = CalculateHeight(root.Left);
        int rightHeight = CalculateHeight(root.Right);

        return Math.Max(leftHeight, rightHeight) + 1;
    }
}
```

### 3. Ağaç Gezinme

```csharp
public class TreeTraversal
{
    // Pre-order traversal
    public void PreOrder(TreeNode root)
    {
        if (root == null)
            return;

        Console.Write(root.Value + " ");
        PreOrder(root.Left);
        PreOrder(root.Right);
    }

    // In-order traversal
    public void InOrder(TreeNode root)
    {
        if (root == null)
            return;

        InOrder(root.Left);
        Console.Write(root.Value + " ");
        InOrder(root.Right);
    }

    // Post-order traversal
    public void PostOrder(TreeNode root)
    {
        if (root == null)
            return;

        PostOrder(root.Left);
        PostOrder(root.Right);
        Console.Write(root.Value + " ");
    }
}
```

## İleri Ağaç Algoritmaları

### 1. Binary Search Tree

```csharp
public class BinarySearchTree
{
    public TreeNode Insert(TreeNode root, int value)
    {
        if (root == null)
            return new TreeNode(value);

        if (value < root.Value)
            root.Left = Insert(root.Left, value);
        else if (value > root.Value)
            root.Right = Insert(root.Right, value);

        return root;
    }

    public TreeNode Search(TreeNode root, int value)
    {
        if (root == null || root.Value == value)
            return root;

        if (value < root.Value)
            return Search(root.Left, value);

        return Search(root.Right, value);
    }
}
```

### 2. Ağaç Dengeleme

```csharp
public class TreeBalancer
{
    public bool IsBalanced(TreeNode root)
    {
        return CheckHeight(root) != -1;
    }

    private int CheckHeight(TreeNode root)
    {
        if (root == null)
            return 0;

        int leftHeight = CheckHeight(root.Left);
        if (leftHeight == -1)
            return -1;

        int rightHeight = CheckHeight(root.Right);
        if (rightHeight == -1)
            return -1;

        if (Math.Abs(leftHeight - rightHeight) > 1)
            return -1;

        return Math.Max(leftHeight, rightHeight) + 1;
    }
}
```

### 3. En Düşük Ortak Ata

```csharp
public class LowestCommonAncestor
{
    public TreeNode FindLCA(TreeNode root, TreeNode p, TreeNode q)
    {
        if (root == null || root == p || root == q)
            return root;

        TreeNode left = FindLCA(root.Left, p, q);
        TreeNode right = FindLCA(root.Right, p, q);

        if (left != null && right != null)
            return root;

        return left ?? right;
    }
}
```

## Performans Analizi

| Algoritma | En İyi Durum | Ortalama Durum | En Kötü Durum | Bellek Kullanımı |
|-----------|-------------|----------------|---------------|------------------|
| Yükseklik | O(n) | O(n) | O(n) | O(h) |
| Gezinme | O(n) | O(n) | O(n) | O(h) |
| BST Arama | O(log n) | O(log n) | O(n) | O(h) |
| BST Ekleme | O(log n) | O(log n) | O(n) | O(h) |

## Best Practices

1. Null kontrollerini yap
2. Denge durumunu kontrol et
3. Bellek kullanımını optimize et
4. Özyinelemeyi dikkatli kullan
5. Hata durumlarını yönet

## Örnek Uygulamalar

1. Veri yapısı implementasyonu
2. Arama algoritmaları
3. Sıralama algoritmaları
4. Veri işleme
5. Sistem tasarımı

## Kaynaklar

- [Tree Data Structure in C#](https://www.geeksforgeeks.org/binary-tree-data-structure/)
- [C# Collections](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic)
- [Data Structures and Algorithms](https://www.tutorialspoint.com/data_structures_algorithms/index.htm) 