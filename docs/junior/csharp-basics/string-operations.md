# String İşlemleri

## Genel Bakış

String işlemleri, metin verilerini manipüle etmek için kullanılan temel operasyonlardır. C#'ta string'ler immutable (değişmez) yapıdadır ve System.String sınıfı tarafından temsil edilir.

## Temel String İşlemleri

1. **String Oluşturma**
   ```csharp
   string str1 = "Merhaba";
   string str2 = new string('a', 5); // "aaaaa"
   string str3 = string.Empty;
   ```

2. **String Birleştirme**
   ```csharp
   string str1 = "Merhaba";
   string str2 = "Dünya";
   
   // + operatörü
   string result1 = str1 + " " + str2;
   
   // String.Concat
   string result2 = string.Concat(str1, " ", str2);
   
   // String.Format
   string result3 = string.Format("{0} {1}", str1, str2);
   
   // String interpolation
   string result4 = $"{str1} {str2}";
   ```

3. **String Karşılaştırma**
   ```csharp
   string str1 = "Merhaba";
   string str2 = "merhaba";
   
   // Case-sensitive karşılaştırma
   bool isEqual1 = str1.Equals(str2);
   
   // Case-insensitive karşılaştırma
   bool isEqual2 = str1.Equals(str2, StringComparison.OrdinalIgnoreCase);
   
   // CompareTo
   int result = str1.CompareTo(str2);
   ```

4. **String Arama**
   ```csharp
   string str = "Merhaba Dünya";
   
   // Contains
   bool contains = str.Contains("Dünya");
   
   // IndexOf
   int index = str.IndexOf("Dünya");
   
   // LastIndexOf
   int lastIndex = str.LastIndexOf("a");
   
   // StartsWith
   bool startsWith = str.StartsWith("Mer");
   
   // EndsWith
   bool endsWith = str.EndsWith("nya");
   ```

## String Manipülasyonu

1. **Substring**
   ```csharp
   string str = "Merhaba Dünya";
   string sub1 = str.Substring(0, 7); // "Merhaba"
   string sub2 = str.Substring(8); // "Dünya"
   ```

2. **Replace**
   ```csharp
   string str = "Merhaba Dünya";
   string newStr = str.Replace("Dünya", "Mars");
   ```

3. **Trim**
   ```csharp
   string str = "  Merhaba  ";
   string trimmed = str.Trim(); // "Merhaba"
   string trimmedStart = str.TrimStart(); // "Merhaba  "
   string trimmedEnd = str.TrimEnd(); // "  Merhaba"
   ```

4. **ToUpper/ToLower**
   ```csharp
   string str = "Merhaba";
   string upper = str.ToUpper(); // "MERHABA"
   string lower = str.ToLower(); // "merhaba"
   ```

## String Formatlama

1. **String.Format**
   ```csharp
   string name = "Ahmet";
   int age = 30;
   string formatted = string.Format("Ad: {0}, Yaş: {1}", name, age);
   ```

2. **String Interpolation**
   ```csharp
   string name = "Ahmet";
   int age = 30;
   string formatted = $"Ad: {name}, Yaş: {age}";
   ```

3. **Composite Formatting**
   ```csharp
   string name = "Ahmet";
   int age = 30;
   string formatted = string.Format("Ad: {0,-10}, Yaş: {1:D3}", name, age);
   ```

## StringBuilder

1. **Temel Kullanım**
   ```csharp
   StringBuilder sb = new StringBuilder();
   sb.Append("Merhaba");
   sb.Append(" ");
   sb.Append("Dünya");
   string result = sb.ToString();
   ```

2. **Performans Optimizasyonu**
   ```csharp
   StringBuilder sb = new StringBuilder(100); // Başlangıç kapasitesi
   for (int i = 0; i < 1000; i++)
   {
       sb.Append(i.ToString());
   }
   ```

## String Split ve Join

1. **Split**
   ```csharp
   string str = "Ahmet,Mehmet,Ali";
   string[] names = str.Split(',');
   
   // Birden fazla ayraç
   string[] parts = str.Split(new char[] { ',', ';' });
   ```

2. **Join**
   ```csharp
   string[] names = { "Ahmet", "Mehmet", "Ali" };
   string joined = string.Join(", ", names);
   ```

## String Validation

1. **Boş Kontrolü**
   ```csharp
   string str = "";
   bool isEmpty = string.IsNullOrEmpty(str);
   bool isWhiteSpace = string.IsNullOrWhiteSpace(str);
   ```

2. **Regex ile Validation**
   ```csharp
   string email = "test@example.com";
   bool isValid = Regex.IsMatch(email, @"^[^@\s]+@[^@\s]+\.[^@\s]+$");
   ```

## Performans Konuları

1. **String Concatenation vs StringBuilder**
   - Küçük string'ler için + operatörü kullan
   - Büyük veya dinamik string'ler için StringBuilder kullan

2. **String Interning**
   ```csharp
   string str1 = "Merhaba";
   string str2 = "Merhaba";
   bool isInterned = string.IsInterned(str1) != null;
   ```

## Güvenlik Konuları

1. **SQL Injection**
   ```csharp
   // Güvensiz
   string query = "SELECT * FROM Users WHERE Name = '" + name + "'";
   
   // Güvenli
   string query = "SELECT * FROM Users WHERE Name = @name";
   ```

2. **XSS Prevention**
   ```csharp
   string userInput = "<script>alert('xss')</script>";
   string safeInput = System.Web.HttpUtility.HtmlEncode(userInput);
   ```

## Mülakat Soruları

1. **String Immutability**
   - String'ler neden immutable'dır?
   - String immutability'nin avantajları ve dezavantajları nelerdir?
   - String değişikliklerinde performans etkisi nasıl olur?

2. **String Karşılaştırma**
   - String.Equals() ve == operatörü arasındaki farklar nelerdir?
   - StringComparison enum'ı ne işe yarar ve hangi durumlarda kullanılır?
   - String.Compare() metodunun dönüş değeri ne anlama gelir?

3. **StringBuilder**
   - StringBuilder ne zaman kullanılmalıdır?
   - StringBuilder'ın kapasitesi nasıl belirlenir?
   - StringBuilder vs String concatenation performans karşılaştırması nasıldır?

4. **String Formatlama**
   - String.Format() ve string interpolation ($"") arasındaki farklar nelerdir?
   - Composite formatting nedir ve nasıl kullanılır?
   - Custom format provider nasıl oluşturulur?

5. **String Manipülasyonu**
   - Substring() metodunun kullanımında dikkat edilmesi gerekenler nelerdir?
   - Replace() metodu case-sensitive midir?
   - Trim() metodunun alternatifleri nelerdir?

6. **String Validation**
   - String.IsNullOrEmpty() ve String.IsNullOrWhiteSpace() arasındaki fark nedir?
   - Regex ile string validation yaparken dikkat edilmesi gerekenler nelerdir?
   - String validation için extension method nasıl yazılır?

7. **String Güvenliği**
   - SQL injection nasıl önlenir?
   - XSS saldırılarına karşı string'ler nasıl korunur?
   - String'lerde güvenli karakter encoding nasıl yapılır?

8. **String Performansı**
   - String interning nedir ve nasıl çalışır?
   - String pooling nedir ve ne zaman kullanılır?
   - String işlemlerinde memory fragmentation nasıl önlenir?

9. **String ve Unicode**
   - String'lerde Unicode karakterler nasıl işlenir?
   - String.Normalize() metodu ne işe yarar?
   - String'lerde encoding dönüşümleri nasıl yapılır?

10. **String ve Koleksiyonlar**
    - String.Split() metodunun alternatifleri nelerdir?
    - String.Join() metodunda performans optimizasyonu nasıl yapılır?
    - String array'leri ile çalışırken dikkat edilmesi gerekenler nelerdir?

## Örnek Kod Soruları

1. **Palindrome Kontrolü**
   ```csharp
   public bool IsPalindrome(string str)
   {
       // Implementasyon
   }
   ```

2. **String Ters Çevirme**
   ```csharp
   public string ReverseString(string str)
   {
       // Implementasyon
   }
   ```

3. **Anagram Kontrolü**
   ```csharp
   public bool AreAnagrams(string str1, string str2)
   {
       // Implementasyon
   }
   ```

4. **String Sıkıştırma**
   ```csharp
   public string CompressString(string str)
   {
       // Örnek: "aaabbbcc" -> "a3b3c2"
   }
   ```

5. **String Karakter Sayımı**
   ```csharp
   public Dictionary<char, int> CountCharacters(string str)
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - Strings](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/strings/)
- [String Manipulation in C#](https://docs.microsoft.com/en-us/dotnet/standard/base-types/manipulating-strings)
- [StringBuilder Class](https://docs.microsoft.com/en-us/dotnet/api/system.text.stringbuilder) 