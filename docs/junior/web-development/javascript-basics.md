# JavaScript Basics

## Giriş

JavaScript, web geliştirmenin temel programlama dilidir. Backend geliştiriciler olarak JavaScript'i anlamak, API entegrasyonu ve frontend-backend iletişimi için kritiktir.

## JavaScript Temel Kavramlar

### Değişkenler ve Veri Tipleri
```javascript
// ES6+ let ve const kullanımı
let userName = "Ahmet";
const userId = 12345;
var oldWay = "eski yöntem"; // ES5

// Veri tipleri
let string = "Metin";
let number = 42;
let boolean = true;
let array = [1, 2, 3, 4, 5];
let object = { name: "Ahmet", age: 25 };
let nullValue = null;
let undefinedValue = undefined;

// Template literals
let message = `Merhaba ${userName}, ID'niz: ${userId}`;
```

### Fonksiyonlar
```javascript
// Function declaration
function greet(name) {
    return `Merhaba ${name}!`;
}

// Function expression
const greetUser = function(name) {
    return `Merhaba ${name}!`;
};

// Arrow function (ES6+)
const greetArrow = (name) => `Merhaba ${name}!`;

// Default parameters
const greetWithDefault = (name = "Misafir") => `Merhaba ${name}!`;

// Rest parameters
const sum = (...numbers) => numbers.reduce((total, num) => total + num, 0);
```

### Array Methods
```javascript
const users = [
    { id: 1, name: "Ahmet", age: 25 },
    { id: 2, name: "Mehmet", age: 30 },
    { id: 3, name: "Ayşe", age: 28 }
];

// map - dönüştürme
const userNames = users.map(user => user.name);

// filter - filtreleme
const youngUsers = users.filter(user => user.age < 30);

// find - bulma
const user = users.find(user => user.id === 2);

// reduce - toplama
const totalAge = users.reduce((sum, user) => sum + user.age, 0);

// forEach - döngü
users.forEach(user => console.log(`${user.name} - ${user.age}`));
```

### Object Destructuring ve Spread
```javascript
const user = {
    id: 1,
    name: "Ahmet",
    email: "ahmet@example.com",
    address: {
        city: "İstanbul",
        country: "Türkiye"
    }
};

// Destructuring
const { name, email, address: { city } } = user;

// Spread operator
const userCopy = { ...user };
const userWithRole = { ...user, role: "admin" };

// Object shorthand
const name2 = "Mehmet";
const age2 = 30;
const person = { name2, age2 }; // { name2: "Mehmet", age2: 30 }
```

## DOM Manipulation

### Element Seçimi
```javascript
// ID ile seçim
const header = document.getElementById('header');

// Class ile seçim
const buttons = document.getElementsByClassName('btn');

// Tag ile seçim
const paragraphs = document.getElementsByTagName('p');

// Modern seçiciler
const element = document.querySelector('.class-name');
const elements = document.querySelectorAll('.class-name');
```

### Element Oluşturma ve Değiştirme
```javascript
// Yeni element oluşturma
const newDiv = document.createElement('div');
newDiv.className = 'new-element';
newDiv.textContent = 'Yeni içerik';

// Element ekleme
document.body.appendChild(newDiv);

// Element değiştirme
const title = document.querySelector('h1');
title.textContent = 'Yeni Başlık';
title.style.color = 'blue';

// Attribute ekleme/çıkarma
const link = document.querySelector('a');
link.setAttribute('target', '_blank');
link.removeAttribute('rel');
```

### Event Handling
```javascript
// Click event
const button = document.querySelector('#submit-btn');
button.addEventListener('click', function(event) {
    event.preventDefault();
    console.log('Buton tıklandı!');
});

// Form submit event
const form = document.querySelector('#user-form');
form.addEventListener('submit', function(event) {
    event.preventDefault();
    
    const formData = new FormData(form);
    const userData = Object.fromEntries(formData);
    
    console.log('Form verisi:', userData);
});

// Input change event
const input = document.querySelector('#username');
input.addEventListener('input', function(event) {
    console.log('Input değeri:', event.target.value);
});
```

## AJAX ve Fetch API

### Fetch API (Modern)
```javascript
// GET request
fetch('/api/users')
    .then(response => {
        if (!response.ok) {
            throw new Error('Network response was not ok');
        }
        return response.json();
    })
    .then(data => {
        console.log('Kullanıcılar:', data);
    })
    .catch(error => {
        console.error('Hata:', error);
    });

// POST request
const createUser = async (userData) => {
    try {
        const response = await fetch('/api/users', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(userData)
        });
        
        if (!response.ok) {
            throw new Error('Kullanıcı oluşturulamadı');
        }
        
        return await response.json();
    } catch (error) {
        console.error('Hata:', error);
        throw error;
    }
};

// Kullanım
createUser({ name: 'Ahmet', email: 'ahmet@example.com' })
    .then(user => console.log('Oluşturulan kullanıcı:', user))
    .catch(error => console.error('Hata:', error));
```

### XMLHttpRequest (Legacy)
```javascript
const xhr = new XMLHttpRequest();
xhr.open('GET', '/api/users', true);

xhr.onreadystatechange = function() {
    if (xhr.readyState === 4) {
        if (xhr.status === 200) {
            const users = JSON.parse(xhr.responseText);
            console.log('Kullanıcılar:', users);
        } else {
            console.error('Hata:', xhr.status);
        }
    }
};

xhr.send();
```

## Error Handling

### Try-Catch
```javascript
try {
    const result = riskyOperation();
    console.log('Sonuç:', result);
} catch (error) {
    console.error('Hata oluştu:', error.message);
    // Hata loglama veya kullanıcıya gösterme
} finally {
    // Her durumda çalışacak kod
    console.log('İşlem tamamlandı');
}
```

### Promise Error Handling
```javascript
fetch('/api/users')
    .then(response => response.json())
    .then(data => {
        // Başarılı işlem
        console.log(data);
    })
    .catch(error => {
        // Hata yakalama
        console.error('Fetch hatası:', error);
    });
```

### Async/Await Error Handling
```javascript
const fetchUsers = async () => {
    try {
        const response = await fetch('/api/users');
        const users = await response.json();
        return users;
    } catch (error) {
        console.error('Kullanıcılar alınamadı:', error);
        throw error;
    }
};
```

## Mülakat Soruları

### Temel Sorular

1. **JavaScript'te var, let ve const arasındaki fark nedir?**
   - **Cevap**: `var` function-scoped, `let` ve `const` block-scoped. `let` değiştirilebilir, `const` değiştirilemez.

2. **JavaScript'te hoisting nedir?**
   - **Cevap**: Function ve variable declaration'ların scope'larının en üstüne taşınması. `var` hoisted edilir, `let` ve `const` edilmez.

3. **Closure nedir?**
   - **Cevap**: Bir fonksiyonun kendi scope'u dışındaki değişkenlere erişebilmesi. Lexical scoping ile oluşur.

4. **Event bubbling ve capturing nedir?**
   - **Cevap**: Event'lerin DOM tree'de yukarı (bubbling) veya aşağı (capturing) yayılması. `addEventListener`'da üçüncü parametre ile kontrol edilir.

5. **Promise nedir?**
   - **Cevap**: Asenkron işlemleri yönetmek için kullanılan yapı. Pending, fulfilled ve rejected durumları vardır.

### Teknik Sorular

1. **JavaScript'te this keyword'ü nasıl çalışır?**
   - **Cevap**: Function'ın nasıl çağrıldığına bağlı olarak değişir. Global scope, object method, constructor, arrow function'da farklı davranır.

2. **Event loop nedir?**
   - **Cevap**: JavaScript'in asenkron işlemleri yönetme mekanizması. Call stack, web APIs, callback queue ve event loop'dan oluşur.

3. **Prototype inheritance nedir?**
   - **Cevap**: JavaScript'te object'ler arasında inheritance sağlayan mekanizma. `Object.create()` ve constructor function'larla kullanılır.

4. **Debouncing ve throttling nedir?**
   - **Cevap**: Event handler'ları optimize etme teknikleri. Debouncing son çağrıyı bekler, throttling belirli aralıklarla çağrı yapar.

5. **Local storage ve session storage arasındaki fark nedir?**
   - **Cevap**: Local storage kalıcı, session storage tarayıcı kapanınca silinir. Her ikisi de 5-10MB kapasiteye sahiptir.

## Best Practices

1. **Kod Organizasyonu**
   - ES6+ syntax kullanın
   - Arrow function'ları tercih edin
   - Template literals kullanın
   - Destructuring ve spread kullanın

2. **Error Handling**
   - Try-catch blokları kullanın
   - Promise rejection'ları yakalayın
   - User-friendly error mesajları gösterin
   - Error logging yapın

3. **Performance**
   - Event delegation kullanın
   - Debouncing/throttling uygulayın
   - DOM manipulation'ı minimize edin
   - Memory leak'leri önleyin

4. **Security**
   - Input validation yapın
   - XSS saldırılarına karşı koruyun
   - CSRF token'ları kullanın
   - HTTPS kullanın

## Kaynaklar

- [MDN JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
- [JavaScript.info](https://javascript.info/)
- [ES6 Features](https://es6-features.org/)
- [You Don't Know JS](https://github.com/getify/You-Dont-Know-JS)
- [JavaScript Best Practices](https://www.w3.org/wiki/JavaScript_best_practices) 