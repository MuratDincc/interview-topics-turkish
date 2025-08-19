# Git Basics

## Giriş

Git, dağıtık versiyon kontrol sistemidir ve modern yazılım geliştirmede vazgeçilmez bir araçtır. Bu dosya, Git'in temel kavramlarını, komutlarını ve en iyi uygulamalarını kapsar.

## Temel Kavramlar

### 1. Git Nedir?
Git, dosyalardaki değişiklikleri takip eden, işbirliği yapılmasını sağlayan dağıtık versiyon kontrol sistemidir.

**Avantajları:**
- Offline çalışma
- Branching ve merging
- Değişiklik geçmişi
- İşbirliği imkanı
- Backup ve recovery

### 2. Git Terminolojisi
```
Repository (Repo)    : Proje klasörü
Working Directory    : Çalışma alanı
Staging Area        : Commit öncesi hazırlık alanı
Commit              : Değişiklikleri kaydetme
Branch              : Paralel geliştirme dalı
Merge               : Dalları birleştirme
```

### 3. Git Workflow
```
Working Directory → Staging Area → Repository
     (add)              (commit)
```

## Temel Git Komutları

### 1. Repository İşlemleri
```bash
# Repository oluşturma
git init

# Uzak repository'yi klonlama
git clone https://github.com/user/repo.git

# Uzak repository ekleme
git remote add origin https://github.com/user/repo.git

# Uzak repository'leri görüntüleme
git remote -v
```

### 2. Temel İşlemler
```bash
# Dosya durumunu kontrol etme
git status

# Dosyaları staging area'ya ekleme
git add filename.txt
git add .                # Tüm dosyalar
git add *.cs            # Belirli uzantıdaki dosyalar

# Commit oluşturma
git commit -m "Commit message"
git commit -am "Add and commit message"  # Add + commit

# Değişiklik geçmişi
git log
git log --oneline       # Kısa format
git log --graph         # Grafik format
```

### 3. Branch İşlemleri
```bash
# Branch'leri listeleme
git branch
git branch -a           # Remote branch'ler dahil

# Yeni branch oluşturma
git branch feature-login
git checkout -b feature-login  # Oluştur ve geç

# Branch değiştirme
git checkout feature-login
git switch feature-login       # Yeni syntax

# Branch silme
git branch -d feature-login    # Safe delete
git branch -D feature-login    # Force delete
```

### 4. Merge ve Rebase
```bash
# Branch'i merge etme
git checkout main
git merge feature-login

# Rebase işlemi
git checkout feature-login
git rebase main

# Merge conflict çözme
git status              # Conflict'li dosyaları görme
# Dosyaları düzenle
git add conflicted-file.txt
git commit
```

## Uzak Repository İşlemleri

### 1. Push ve Pull
```bash
# Uzak repository'ye gönderme
git push origin main
git push -u origin feature-login  # Upstream set

# Uzak repository'den çekme
git pull origin main
git pull                # Default branch

# Fetch işlemi
git fetch origin        # Download but don't merge
git merge origin/main   # Manual merge
```

### 2. Remote Branch İşlemleri
```bash
# Remote branch'leri görme
git branch -r

# Remote branch'i local'e getirme
git checkout -b feature-remote origin/feature-remote

# Remote branch'i silme
git push origin --delete feature-remote
```

## .gitignore Dosyası

### 1. .gitignore Örnekleri
```gitignore
# .NET Core
bin/
obj/
*.user
*.suo
.vs/
.vscode/

# Build results
[Dd]ebug/
[Rr]elease/
x64/
x86/

# NuGet packages
*.nupkg
packages/

# User-specific files
*.userprefs
*.DotSettings

# OS generated files
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Database
*.db
*.sqlite
```

### 2. .gitignore Komutları
```bash
# .gitignore dosyası oluşturma
touch .gitignore

# Zaten track edilen dosyayı ignore etme
git rm --cached filename.txt
git commit -m "Remove tracked file"

# Global .gitignore
git config --global core.excludesfile ~/.gitignore_global
```

## Git Configuration

### 1. User Configuration
```bash
# Global user ayarları
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Repository-specific ayarlar
git config user.name "Work Name"
git config user.email "work.email@company.com"

# Configuration görüntüleme
git config --list
git config user.name
```

### 2. Diğer Ayarlar
```bash
# Default editor
git config --global core.editor "code --wait"

# Line ending ayarları
git config --global core.autocrlf true    # Windows
git config --global core.autocrlf input   # Mac/Linux

# Default branch name
git config --global init.defaultBranch main

# Alias oluşturma
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
```

## Pratik Örnekler

### 1. Yeni Proje Başlatma
```bash
# Local repository oluşturma
mkdir MyProject
cd MyProject
git init

# İlk commit
echo "# My Project" > README.md
git add README.md
git commit -m "Initial commit"

# Remote repository ekleme
git remote add origin https://github.com/username/MyProject.git
git push -u origin main
```

### 2. Feature Branch Workflow
```bash
# Ana daldan yeni branch oluşturma
git checkout main
git pull origin main
git checkout -b feature/user-authentication

# Değişiklik yapma ve commit
# ... kod yazma ...
git add .
git commit -m "Add user authentication feature"

# Remote'a push etme
git push -u origin feature/user-authentication

# Pull request oluşturma (GitHub/GitLab'da)
# Merge sonrası cleanup
git checkout main
git pull origin main
git branch -d feature/user-authentication
```

### 3. Hotfix Workflow
```bash
# Acil düzeltme için branch
git checkout main
git checkout -b hotfix/critical-bug

# Düzeltme yapma
# ... bug fix ...
git add .
git commit -m "Fix critical bug in payment process"

# Main'e merge
git checkout main
git merge hotfix/critical-bug
git push origin main

# Cleanup
git branch -d hotfix/critical-bug
```

## Mülakat Soruları

### Temel Sorular

1. **Git nedir ve neden kullanılır?**
   - **Cevap**: Dağıtık versiyon kontrol sistemi, kod değişikliklerini takip eder.

2. **Working Directory, Staging Area ve Repository arasındaki fark nedir?**
   - **Cevap**: Working Directory çalışma alanı, Staging Area commit öncesi hazırlık, Repository kalıcı depolama.

3. **git add ve git commit arasındaki fark nedir?**
   - **Cevap**: git add staging area'ya ekler, git commit repository'ye kaydeder.

4. **Branch nedir ve neden kullanılır?**
   - **Cevap**: Paralel geliştirme dalı, feature geliştirme ve izolasyon için.

5. **Merge ile rebase arasındaki fark nedir?**
   - **Cevap**: Merge dalları birleştirir, rebase commit history'yi yeniden düzenler.

### Teknik Sorular

1. **Merge conflict nasıl çözülür?**
   - **Cevap**: Conflict'li dosyalar düzenlenir, git add ile eklenir, commit yapılır.

2. **git pull ile git fetch arasındaki fark nedir?**
   - **Cevap**: git pull fetch + merge yapar, git fetch sadece download eder.

3. **Detached HEAD state nedir?**
   - **Cevap**: Belirli bir commit'te olup branch'te olmama durumu.

4. **git reset ile git revert arasındaki fark nedir?**
   - **Cevap**: reset history'yi değiştirir, revert yeni commit ile geri alır.

## Best Practices

### 1. **Commit Messages**
- Açık ve anlamlı mesajlar yazın
- İmperatif mood kullanın ("Add" not "Added")
- 50 karakter limit başlık için
- Detay gerekirse boş satır sonrası açıklama

### 2. **Branching Strategy**
- Feature branch'ler kullanın
- Meaningful branch isimleri (feature/login, bugfix/payment)
- Regular merge/rebase yapın
- Dead branch'leri temizleyin

### 3. **Repository Hygiene**
- .gitignore dosyasını düzgün kullanın
- Sensitive data commit etmeyin
- Binary dosyaları minimize edin
- Regular cleanup yapın

### 4. **Collaboration**
- Pull request'ler kullanın
- Code review yapın
- Conflict'leri hızlı çözün
- Documentation güncel tutun

## Kaynaklar

- [Git Official Documentation](https://git-scm.com/doc)
- [Pro Git Book](https://git-scm.com/book)
- [GitHub Git Handbook](https://guides.github.com/introduction/git-handbook/)
- [Atlassian Git Tutorials](https://www.atlassian.com/git/tutorials)
- [Git Cheat Sheet](https://education.github.com/git-cheat-sheet-education.pdf)
