# HTML & CSS Basics

## Giriş

HTML (HyperText Markup Language) ve CSS (Cascading Style Sheets), web geliştirmenin temel yapı taşlarıdır. Backend geliştiriciler olarak, bu teknolojileri anlamak API tasarımı ve frontend entegrasyonu için kritiktir.

## HTML Temelleri

### HTML Yapısı
```html
<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Sayfa Başlığı</title>
</head>
<body>
    <header>
        <h1>Ana Başlık</h1>
        <nav>
            <ul>
                <li><a href="#home">Ana Sayfa</a></li>
                <li><a href="#about">Hakkında</a></li>
            </ul>
        </nav>
    </header>
    
    <main>
        <section>
            <h2>Bölüm Başlığı</h2>
            <p>Paragraf metni</p>
        </section>
    </main>
    
    <footer>
        <p>&copy; 2024 Tüm hakları saklıdır</p>
    </footer>
</body>
</html>
```

### Semantik HTML Elementleri
- `<header>`: Sayfa başlığı ve navigasyon
- `<nav>`: Navigasyon menüsü
- `<main>`: Ana içerik
- `<section>`: İçerik bölümü
- `<article>`: Bağımsız içerik
- `<aside>`: Yan içerik
- `<footer>`: Sayfa alt bilgisi

### Form Elementleri
```html
<form action="/api/users" method="POST">
    <div>
        <label for="username">Kullanıcı Adı:</label>
        <input type="text" id="username" name="username" required>
    </div>
    
    <div>
        <label for="email">E-posta:</label>
        <input type="email" id="email" name="email" required>
    </div>
    
    <div>
        <label for="password">Şifre:</label>
        <input type="password" id="password" name="password" required>
    </div>
    
    <button type="submit">Kaydet</button>
</form>
```

## CSS Temelleri

### CSS Seçicileri
```css
/* Element seçici */
h1 {
    color: blue;
    font-size: 24px;
}

/* Class seçici */
.button {
    background-color: #007bff;
    color: white;
    padding: 10px 20px;
    border: none;
    border-radius: 4px;
}

/* ID seçici */
#header {
    background-color: #f8f9fa;
    padding: 20px;
}

/* Descendant seçici */
nav ul li {
    display: inline;
    margin-right: 15px;
}
```

### CSS Box Model
```css
.box {
    width: 200px;
    height: 100px;
    padding: 20px;
    border: 2px solid #333;
    margin: 10px;
    box-sizing: border-box;
}
```

### Flexbox Layout
```css
.container {
    display: flex;
    justify-content: space-between;
    align-items: center;
    flex-wrap: wrap;
}

.item {
    flex: 1;
    margin: 10px;
}
```

### Grid Layout
```css
.grid-container {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-gap: 20px;
}

.grid-item {
    background-color: #f0f0f0;
    padding: 20px;
    text-align: center;
}
```

## Responsive Design

### Media Queries
```css
/* Mobil cihazlar */
@media (max-width: 768px) {
    .container {
        flex-direction: column;
    }
    
    .grid-container {
        grid-template-columns: 1fr;
    }
}

/* Tablet cihazlar */
@media (min-width: 769px) and (max-width: 1024px) {
    .grid-container {
        grid-template-columns: repeat(2, 1fr);
    }
}
```

### Viewport Meta Tag
```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

## CSS Framework'leri

### Bootstrap
```html
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">

<div class="container">
    <div class="row">
        <div class="col-md-6">
            <div class="card">
                <div class="card-body">
                    <h5 class="card-title">Kart Başlığı</h5>
                    <p class="card-text">Kart içeriği</p>
                    <button class="btn btn-primary">Buton</button>
                </div>
            </div>
        </div>
    </div>
</div>
```

## Mülakat Soruları

### Temel Sorular

1. **HTML'de semantic elementler neden önemlidir?**
   - **Cevap**: SEO, accessibility ve kod okunabilirliği için önemlidir. Arama motorları ve screen reader'lar sayfa yapısını daha iyi anlar.

2. **CSS'de specificity nedir?**
   - **Cevap**: CSS kurallarının hangi sırayla uygulanacağını belirleyen kural. ID > Class > Element sırasında öncelik vardır.

3. **Responsive design nedir?**
   - **Cevap**: Farklı ekran boyutlarında uyumlu görünüm sağlayan tasarım yaklaşımı. Media queries ve flexible layout kullanır.

4. **CSS Box Model nedir?**
   - **Cevap**: Content, padding, border ve margin'den oluşan element yapısı. `box-sizing: border-box` ile padding ve border dahil edilir.

5. **Flexbox ve Grid arasındaki fark nedir?**
   - **Cevap**: Flexbox tek boyutlu (row veya column), Grid iki boyutlu layout için kullanılır. Flexbox daha esnek, Grid daha yapılandırılmış.

### Teknik Sorular

1. **CSS'de z-index nasıl çalışır?**
   - **Cevap**: Elementlerin katman sırasını belirler. Yüksek değer önde görünür. Stacking context'e bağlıdır.

2. **CSS'de inheritance nedir?**
   - **Cevap**: Child elementlerin parent elementlerden CSS özelliklerini alması. Font, color gibi özellikler inherit edilir.

3. **CSS'de pseudo-class ve pseudo-element nedir?**
   - **Cevap**: Pseudo-class (:hover, :focus) durumları, pseudo-element (::before, ::after) içerik eklemeyi sağlar.

4. **CSS'de vendor prefix nedir?**
   - **Cevap**: Tarayıcı uyumluluğu için CSS özelliklerinin önüne eklenen prefix'ler (-webkit-, -moz- gibi).

5. **CSS'de BEM metodolojisi nedir?**
   - **Cevap**: Block__Element--Modifier yapısı. CSS class'larını organize etmek için kullanılan naming convention.

## Best Practices

1. **HTML**
   - Semantic elementler kullanın
   - Alt text ekleyin
   - Form validation yapın
   - Accessibility standartlarına uyun

2. **CSS**
   - CSS reset kullanın
   - Mobile-first yaklaşım benimseyin
   - CSS variables kullanın
   - Performance için kritik CSS'i inline yapın

3. **Responsive Design**
   - Breakpoint'leri mantıklı seçin
   - Touch-friendly tasarım yapın
   - Performance'ı optimize edin
   - Testing yapın

## Kaynaklar

- [MDN HTML](https://developer.mozilla.org/en-US/docs/Web/HTML)
- [MDN CSS](https://developer.mozilla.org/en-US/docs/Web/CSS)
- [CSS Grid](https://css-tricks.com/snippets/css/complete-guide-grid/)
- [Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)
- [Bootstrap Documentation](https://getbootstrap.com/docs/) 