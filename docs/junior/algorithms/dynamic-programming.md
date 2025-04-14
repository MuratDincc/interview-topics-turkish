# Dynamic Programming

## Genel Bakış

Dinamik programlama, karmaşık problemleri daha küçük alt problemlere bölerek çözen bir algoritma tasarım tekniğidir. Bu bölümde, dinamik programlama yaklaşımını ve C# implementasyonlarını inceleyeceğiz.

## Temel Dinamik Programlama Algoritmaları

### 1. Fibonacci Sayıları

```csharp
public class FibonacciDP
{
    public int CalculateFibonacci(int n)
    {
        if (n <= 1)
            return n;

        int[] dp = new int[n + 1];
        dp[0] = 0;
        dp[1] = 1;

        for (int i = 2; i <= n; i++)
        {
            dp[i] = dp[i - 1] + dp[i - 2];
        }

        return dp[n];
    }
}
```

### 2. En Uzun Artan Alt Dizi

```csharp
public class LongestIncreasingSubsequence
{
    public int FindLIS(int[] nums)
    {
        if (nums == null || nums.Length == 0)
            return 0;

        int[] dp = new int[nums.Length];
        Array.Fill(dp, 1);

        for (int i = 1; i < nums.Length; i++)
        {
            for (int j = 0; j < i; j++)
            {
                if (nums[i] > nums[j])
                {
                    dp[i] = Math.Max(dp[i], dp[j] + 1);
                }
            }
        }

        return dp.Max();
    }
}
```

### 3. 0-1 Knapsack Problemi

```csharp
public class Knapsack
{
    public int MaxValue(int[] weights, int[] values, int capacity)
    {
        int n = weights.Length;
        int[,] dp = new int[n + 1, capacity + 1];

        for (int i = 1; i <= n; i++)
        {
            for (int w = 1; w <= capacity; w++)
            {
                if (weights[i - 1] <= w)
                {
                    dp[i, w] = Math.Max(
                        values[i - 1] + dp[i - 1, w - weights[i - 1]],
                        dp[i - 1, w]
                    );
                }
                else
                {
                    dp[i, w] = dp[i - 1, w];
                }
            }
        }

        return dp[n, capacity];
    }
}
```

## Dinamik Programlama Teknikleri

### 1. Memoization

```csharp
public class MemoizationExample
{
    private Dictionary<int, int> memo = new Dictionary<int, int>();

    public int Calculate(int n)
    {
        if (memo.ContainsKey(n))
            return memo[n];

        int result;
        if (n <= 1)
            result = n;
        else
            result = Calculate(n - 1) + Calculate(n - 2);

        memo[n] = result;
        return result;
    }
}
```

### 2. Tabulation

```csharp
public class TabulationExample
{
    public int Calculate(int n)
    {
        int[] table = new int[n + 1];
        table[0] = 0;
        table[1] = 1;

        for (int i = 2; i <= n; i++)
        {
            table[i] = table[i - 1] + table[i - 2];
        }

        return table[n];
    }
}
```

## Performans Analizi

| Algoritma | En İyi Durum | Ortalama Durum | En Kötü Durum | Bellek Kullanımı |
|-----------|-------------|----------------|---------------|------------------|
| Fibonacci | O(n) | O(n) | O(n) | O(n) |
| LIS | O(n²) | O(n²) | O(n²) | O(n) |
| Knapsack | O(nW) | O(nW) | O(nW) | O(nW) |

## Best Practices

1. Alt problemleri tanımla
2. Özyinelemeli formülü bul
3. Tablo boyutunu optimize et
4. Bellek kullanımını dikkate al
5. Hata durumlarını yönet

## Örnek Uygulamalar

1. En kısa yol problemleri
2. En uzun ortak alt dizi
3. Para üstü verme
4. Matris çarpımı
5. Dizi bölme

## Kaynaklar

- [Dynamic Programming Tutorial](https://www.geeksforgeeks.org/dynamic-programming/)
- [C# Dynamic Programming Examples](https://www.c-sharpcorner.com/article/dynamic-programming-in-c-sharp/)
- [Dynamic Programming Patterns](https://leetcode.com/discuss/general-discussion/458695/dynamic-programming-patterns) 