# String Algorithms

## Genel Bakış

String algoritmaları, metin işleme ve metin arama gibi işlemlerde kullanılan temel algoritmalardır. Bu bölümde, string işlemleri için kullanılan temel algoritmaları ve bunların C# implementasyonlarını inceleyeceğiz.

## Temel String İşlemleri

### 1. Palindrome Kontrolü

```csharp
public class PalindromeChecker
{
    public bool IsPalindrome(string text)
    {
        if (string.IsNullOrEmpty(text))
            return true;

        int left = 0;
        int right = text.Length - 1;

        while (left < right)
        {
            if (char.ToLower(text[left]) != char.ToLower(text[right]))
                return false;

            left++;
            right--;
        }

        return true;
    }
}
```

### 2. Anagram Kontrolü

```csharp
public class AnagramChecker
{
    public bool AreAnagrams(string s1, string s2)
    {
        if (s1.Length != s2.Length)
            return false;

        var charCount = new int[256];

        foreach (char c in s1)
            charCount[c]++;

        foreach (char c in s2)
        {
            charCount[c]--;
            if (charCount[c] < 0)
                return false;
        }

        return true;
    }
}
```

### 3. En Uzun Palindromik Alt String

```csharp
public class LongestPalindromicSubstringFinder
{
    public string FindLongestPalindrome(string s)
    {
        if (string.IsNullOrEmpty(s))
            return string.Empty;

        int start = 0;
        int maxLength = 1;

        for (int i = 0; i < s.Length; i++)
        {
            int len1 = ExpandAroundCenter(s, i, i);
            int len2 = ExpandAroundCenter(s, i, i + 1);
            int len = Math.Max(len1, len2);

            if (len > maxLength)
            {
                start = i - (len - 1) / 2;
                maxLength = len;
            }
        }

        return s.Substring(start, maxLength);
    }

    private int ExpandAroundCenter(string s, int left, int right)
    {
        while (left >= 0 && right < s.Length && s[left] == s[right])
        {
            left--;
            right++;
        }
        return right - left - 1;
    }
}
```

## String Arama Algoritmaları

### 1. Naive String Search

```csharp
public class NaiveStringSearch
{
    public List<int> Search(string text, string pattern)
    {
        var positions = new List<int>();
        int n = text.Length;
        int m = pattern.Length;

        for (int i = 0; i <= n - m; i++)
        {
            int j;
            for (j = 0; j < m; j++)
            {
                if (text[i + j] != pattern[j])
                    break;
            }

            if (j == m)
                positions.Add(i);
        }

        return positions;
    }
}
```

### 2. KMP Algorithm

```csharp
public class KMPSearch
{
    public List<int> Search(string text, string pattern)
    {
        var positions = new List<int>();
        int[] lps = ComputeLPSArray(pattern);
        int i = 0;
        int j = 0;

        while (i < text.Length)
        {
            if (pattern[j] == text[i])
            {
                i++;
                j++;
            }

            if (j == pattern.Length)
            {
                positions.Add(i - j);
                j = lps[j - 1];
            }
            else if (i < text.Length && pattern[j] != text[i])
            {
                if (j != 0)
                    j = lps[j - 1];
                else
                    i++;
            }
        }

        return positions;
    }

    private int[] ComputeLPSArray(string pattern)
    {
        int[] lps = new int[pattern.Length];
        int len = 0;
        int i = 1;

        while (i < pattern.Length)
        {
            if (pattern[i] == pattern[len])
            {
                len++;
                lps[i] = len;
                i++;
            }
            else
            {
                if (len != 0)
                    len = lps[len - 1];
                else
                {
                    lps[i] = 0;
                    i++;
                }
            }
        }

        return lps;
    }
}
```

## Performans Analizi

| Algoritma | En İyi Durum | Ortalama Durum | En Kötü Durum | Bellek Kullanımı |
|-----------|-------------|----------------|---------------|------------------|
| Naive Search | O(n) | O(n*m) | O(n*m) | O(1) |
| KMP | O(n) | O(n) | O(n) | O(m) |
| Palindrome | O(n) | O(n) | O(n) | O(1) |
| Anagram | O(n) | O(n) | O(n) | O(1) |

## Best Practices

1. String null kontrolü yap
2. Büyük/küçük harf duyarlılığını dikkate al
3. Unicode karakterleri dikkate al
4. String immutability'yi göz önünde bulundur
5. StringBuilder kullanımını değerlendir

## Örnek Uygulamalar

1. Metin arama
2. Metin değiştirme
3. Metin karşılaştırma
4. Metin sıkıştırma
5. Metin şifreleme

## Kaynaklar

- [String Class Documentation](https://docs.microsoft.com/en-us/dotnet/api/system.string)
- [C# String Tutorial](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/strings/)
- [String Algorithms in C#](https://www.geeksforgeeks.org/string-data-structure/) 