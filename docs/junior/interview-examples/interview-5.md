# Mülakat Örneği 5 - Algoritma ve Veri Yapıları

## 1. Big O notasyonu nedir? Neden önemlidir?
**Cevap:** Big O notasyonu, bir algoritmanın zaman ve bellek karmaşıklığını ifade etmek için kullanılan matematiksel bir gösterimdir. Algoritmanın performansını ve ölçeklenebilirliğini değerlendirmek için önemlidir.

## 2. Array ve List arasındaki farklar nelerdir?
**Cevap:** Array sabit boyutludur ve bellek üzerinde ardışık olarak saklanır. List ise dinamik boyutludur ve gerektiğinde genişleyebilir. List, array'e göre daha esnek ama biraz daha yavaştır.

## 3. Stack ve Queue veri yapıları arasındaki farklar nelerdir?
**Cevap:** Stack LIFO (Last In First Out), Queue ise FIFO (First In First Out) prensibine göre çalışır. Stack'te son eklenen eleman ilk çıkar, Queue'da ise ilk eklenen eleman ilk çıkar.

## 4. Binary Search Tree nedir? Avantajları nelerdir?
**Cevap:** Binary Search Tree, her düğümün en fazla iki çocuğu olan ve sol alt ağaçtaki değerlerin kök düğümden küçük, sağ alt ağaçtaki değerlerin büyük olduğu bir ağaç yapısıdır. Arama, ekleme ve silme işlemleri O(log n) karmaşıklığında yapılabilir.

## 5. Hash Table nedir? Nasıl çalışır?
**Cevap:** Hash Table, anahtar-değer çiftlerini saklamak için kullanılan bir veri yapısıdır. Hash fonksiyonu kullanarak anahtarları indekslere dönüştürür ve çakışmaları (collision) çözmek için çeşitli yöntemler kullanır.

## 6. Bubble Sort, Quick Sort ve Merge Sort arasındaki farklar nelerdir?
**Cevap:** Bubble Sort O(n²) karmaşıklığında basit ama yavaş bir algoritmadır. Quick Sort ortalama O(n log n) karmaşıklığında ve yerinde sıralama yapar. Merge Sort O(n log n) karmaşıklığında ama ek bellek kullanır.

## 7. Recursion nedir? Ne zaman kullanılmalıdır?
**Cevap:** Recursion, bir fonksiyonun kendisini çağırmasıdır. Tree traversal, factorial hesaplama gibi doğal olarak recursive olan problemlerde kullanılmalıdır. Stack overflow riskine dikkat edilmelidir.

## 8. Graph veri yapısı nedir? Nasıl temsil edilir?
**Cevap:** Graph, düğümler ve bu düğümleri birbirine bağlayan kenarlardan oluşan bir veri yapısıdır. Adjacency matrix veya adjacency list kullanılarak temsil edilebilir.

## 9. Breadth First Search (BFS) ve Depth First Search (DFS) arasındaki farklar nelerdir?
**Cevap:** BFS seviye seviye, DFS ise dal dal arama yapar. BFS Queue, DFS ise Stack kullanır. BFS en kısa yolu bulmak için, DFS ise topolojik sıralama gibi işlemler için uygundur.

## 10. Dynamic Programming nedir? Ne zaman kullanılır?
**Cevap:** Dynamic Programming, karmaşık problemleri alt problemlere bölerek çözen bir yaklaşımdır. Alt problemlerin çözümlerini saklayarak tekrar hesaplamayı önler. Fibonacci, Knapsack gibi problemlerde kullanılır.

## 11. Linked List nedir? Avantajları ve dezavantajları nelerdir?
**Cevap:** Linked List, düğümlerden oluşan ve her düğümün bir sonraki düğümü işaret ettiği bir veri yapısıdır. Ekleme ve silme işlemleri O(1) karmaşıklığında yapılabilir ama rastgele erişim O(n) karmaşıklığındadır.

## 12. Heap veri yapısı nedir? Ne için kullanılır?
**Cevap:** Heap, öncelik kuyruğu (priority queue) implementasyonu için kullanılan bir ağaç yapısıdır. Min heap ve max heap olmak üzere iki türü vardır. En yüksek veya en düşük öncelikli elemana hızlı erişim sağlar.

## 13. Time Complexity ve Space Complexity arasındaki fark nedir?
**Cevap:** Time Complexity, bir algoritmanın çalışma süresinin giriş boyutuna göre nasıl değiştiğini, Space Complexity ise algoritmanın ne kadar bellek kullandığını ifade eder.

## 14. Greedy Algorithm nedir? Ne zaman kullanılır?
**Cevap:** Greedy Algorithm, her adımda en iyi görünen seçimi yapan bir yaklaşımdır. Her zaman optimal çözüm vermeyebilir ama hızlıdır. Minimum Spanning Tree, Huffman Coding gibi problemlerde kullanılır.

## 15. Binary Search nedir? Nasıl çalışır?
**Cevap:** Binary Search, sıralı bir dizide arama yapan O(log n) karmaşıklığında bir algoritmadır. Diziyi ortadan ikiye bölerek arama yapar ve her adımda arama alanını yarıya indirir.

## 16. Trie veri yapısı nedir? Ne için kullanılır?
**Cevap:** Trie, string'leri saklamak için kullanılan bir ağaç yapısıdır. Her düğüm bir karakteri temsil eder. Otomatik tamamlama, sözlük uygulamaları gibi alanlarda kullanılır.

## 17. Red-Black Tree nedir? Ne için kullanılır?
**Cevap:** Red-Black Tree, kendini dengeleyen bir binary search tree'dür. Her işlem sonrası ağacı dengede tutar. Java'nın TreeMap, C++'ın map gibi veri yapılarında kullanılır.

## 18. Dijkstra's Algorithm nedir? Nasıl çalışır?
**Cevap:** Dijkstra's Algorithm, bir graph'ta iki düğüm arasındaki en kısa yolu bulan bir algoritmadır. Greedy yaklaşımı kullanır ve negatif ağırlıklı kenarlarla çalışmaz.

## 19. Kruskal's ve Prim's Algorithm arasındaki farklar nelerdir?
**Cevap:** Her iki algoritma da minimum spanning tree bulmak için kullanılır. Kruskal's kenarları sıralayarak çalışır, Prim's ise düğümleri ekleyerek çalışır. Kruskal's sparse graph'larda, Prim's dense graph'larda daha verimlidir.

## 20. Memoization nedir? Ne zaman kullanılır?
**Cevap:** Memoization, fonksiyon çağrılarının sonuçlarını saklayarak tekrar hesaplamayı önleyen bir optimizasyon tekniğidir. Recursive fonksiyonlarda ve dynamic programming'de sıkça kullanılır. 