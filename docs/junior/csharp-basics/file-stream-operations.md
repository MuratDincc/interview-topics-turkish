# File ve Stream İşlemleri

## Genel Bakış

File ve Stream işlemleri, dosya sisteminde veri okuma ve yazma işlemlerini gerçekleştirmek için kullanılan temel operasyonlardır. C#'ta System.IO namespace'i altında bulunan sınıflar kullanılarak dosya ve stream işlemleri yapılır.

## File İşlemleri

1. **Dosya Oluşturma ve Silme**
   ```csharp
   // Dosya oluşturma
   string path = "test.txt";
   File.Create(path);
   
   // Dosya silme
   File.Delete(path);
   
   // Dosya var mı kontrolü
   bool exists = File.Exists(path);
   ```

2. **Dosya Kopyalama ve Taşıma**
   ```csharp
   string sourcePath = "source.txt";
   string destPath = "dest.txt";
   
   // Dosya kopyalama
   File.Copy(sourcePath, destPath);
   
   // Dosya taşıma
   File.Move(sourcePath, destPath);
   ```

3. **Dosya Özellikleri**
   ```csharp
   string path = "test.txt";
   
   // Oluşturma tarihi
   DateTime creationTime = File.GetCreationTime(path);
   
   // Son erişim tarihi
   DateTime lastAccessTime = File.GetLastAccessTime(path);
   
   // Son değişiklik tarihi
   DateTime lastWriteTime = File.GetLastWriteTime(path);
   
   // Dosya boyutu
   long fileSize = new FileInfo(path).Length;
   ```

4. **Dosya Okuma ve Yazma**
   ```csharp
   string path = "test.txt";
   
   // Dosyaya yazma
   File.WriteAllText(path, "Merhaba Dünya");
   
   // Dosyadan okuma
   string content = File.ReadAllText(path);
   
   // Satır satır okuma
   string[] lines = File.ReadAllLines(path);
   
   // Byte array olarak okuma
   byte[] bytes = File.ReadAllBytes(path);
   ```

## Stream İşlemleri

1. **FileStream Kullanımı**
   ```csharp
   string path = "test.txt";
   
   // Stream oluşturma
   using (FileStream stream = new FileStream(path, FileMode.OpenOrCreate))
   {
       // Byte array yazma
       byte[] data = Encoding.UTF8.GetBytes("Merhaba Dünya");
       stream.Write(data, 0, data.Length);
       
       // Byte array okuma
       byte[] buffer = new byte[stream.Length];
       stream.Read(buffer, 0, buffer.Length);
       string content = Encoding.UTF8.GetString(buffer);
   }
   ```

2. **StreamReader ve StreamWriter**
   ```csharp
   string path = "test.txt";
   
   // StreamWriter ile yazma
   using (StreamWriter writer = new StreamWriter(path))
   {
       writer.WriteLine("Satır 1");
       writer.WriteLine("Satır 2");
   }
   
   // StreamReader ile okuma
   using (StreamReader reader = new StreamReader(path))
   {
       string line;
       while ((line = reader.ReadLine()) != null)
       {
           Console.WriteLine(line);
       }
   }
   ```

3. **MemoryStream Kullanımı**
   ```csharp
   using (MemoryStream stream = new MemoryStream())
   {
       // Stream'e yazma
       byte[] data = Encoding.UTF8.GetBytes("Merhaba Dünya");
       stream.Write(data, 0, data.Length);
       
       // Stream'den okuma
       stream.Position = 0;
       byte[] buffer = new byte[stream.Length];
       stream.Read(buffer, 0, buffer.Length);
       string content = Encoding.UTF8.GetString(buffer);
   }
   ```

## Asenkron Dosya İşlemleri

1. **Asenkron Okuma ve Yazma**
   ```csharp
   string path = "test.txt";
   
   // Asenkron yazma
   await File.WriteAllTextAsync(path, "Merhaba Dünya");
   
   // Asenkron okuma
   string content = await File.ReadAllTextAsync(path);
   ```

2. **Asenkron Stream İşlemleri**
   ```csharp
   string path = "test.txt";
   
   using (FileStream stream = new FileStream(path, FileMode.OpenOrCreate))
   {
       // Asenkron yazma
       byte[] data = Encoding.UTF8.GetBytes("Merhaba Dünya");
       await stream.WriteAsync(data, 0, data.Length);
       
       // Asenkron okuma
       byte[] buffer = new byte[stream.Length];
       await stream.ReadAsync(buffer, 0, buffer.Length);
       string content = Encoding.UTF8.GetString(buffer);
   }
   ```

## Dosya Güvenliği

1. **Dosya İzinleri**
   ```csharp
   string path = "test.txt";
   
   // Dosya izinlerini alma
   FileSecurity security = File.GetAccessControl(path);
   
   // Dosya izinlerini ayarlama
   File.SetAccessControl(path, security);
   ```

2. **Dosya Şifreleme**
   ```csharp
   string path = "test.txt";
   
   // Dosya şifreleme
   File.Encrypt(path);
   
   // Dosya şifre çözme
   File.Decrypt(path);
   ```

## Mülakat Soruları

1. **File İşlemleri**
   - File ve FileInfo sınıfları arasındaki farklar nelerdir?
   - File.Exists() ve FileInfo.Exists arasındaki fark nedir?
   - Dosya işlemlerinde exception handling nasıl yapılır?

2. **Stream İşlemleri**
   - Stream nedir ve ne işe yarar?
   - FileStream, MemoryStream ve NetworkStream arasındaki farklar nelerdir?
   - Stream'lerde buffer kullanımının önemi nedir?

3. **Asenkron İşlemler**
   - Asenkron dosya işlemleri ne zaman kullanılmalıdır?
   - Stream'lerde asenkron işlemler nasıl yapılır?
   - Asenkron işlemlerde exception handling nasıl yapılır?

4. **Performans Optimizasyonu**
   - Büyük dosyaların okunmasında performans optimizasyonu nasıl yapılır?
   - Stream'lerde buffer boyutu nasıl belirlenir?
   - Dosya işlemlerinde memory kullanımı nasıl optimize edilir?

5. **Güvenlik**
   - Dosya işlemlerinde güvenlik nasıl sağlanır?
   - Dosya izinleri nasıl yönetilir?
   - Dosya şifreleme nasıl yapılır?

6. **Dosya Formatları**
   - Farklı dosya formatları (txt, csv, json vb.) nasıl işlenir?
   - Binary dosyalar nasıl okunur ve yazılır?
   - Dosya formatı dönüşümleri nasıl yapılır?

7. **Dosya Sistemi**
   - Dosya sistemi işlemleri nasıl yapılır?
   - Klasör işlemleri nasıl yapılır?
   - Dosya sistemi izinleri nasıl kontrol edilir?

8. **Hata Yönetimi**
   - Dosya işlemlerinde karşılaşılan hatalar nelerdir?
   - FileNotFoundException nasıl yönetilir?
   - IOException nasıl yönetilir?

9. **Resource Yönetimi**
   - Stream'lerde using bloğu neden önemlidir?
   - Dispose pattern nedir ve nasıl uygulanır?
   - Resource leak nasıl önlenir?

10. **Best Practices**
    - Dosya işlemlerinde best practices nelerdir?
    - Stream kullanımında dikkat edilmesi gerekenler nelerdir?
    - Dosya işlemlerinde performans ve güvenlik nasıl dengelenir?

## Örnek Kod Soruları

1. **Dosya Kopyalama**
   ```csharp
   public void CopyFile(string sourcePath, string destPath)
   {
       // Implementasyon
   }
   ```

2. **Dosya Boyutu Kontrolü**
   ```csharp
   public bool IsFileSizeValid(string path, long maxSize)
   {
       // Implementasyon
   }
   ```

3. **Dosya İçeriği Arama**
   ```csharp
   public bool SearchInFile(string path, string searchText)
   {
       // Implementasyon
   }
   ```

4. **Dosya Şifreleme**
   ```csharp
   public void EncryptFile(string path, string password)
   {
       // Implementasyon
   }
   ```

5. **Büyük Dosya Okuma**
   ```csharp
   public async Task<string> ReadLargeFileAsync(string path)
   {
       // Implementasyon
   }
   ```

## Kaynaklar

- [Microsoft Docs - File and Stream I/O](https://docs.microsoft.com/en-us/dotnet/standard/io/)
- [File Class](https://docs.microsoft.com/en-us/dotnet/api/system.io.file)
- [Stream Class](https://docs.microsoft.com/en-us/dotnet/api/system.io.stream) 