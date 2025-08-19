# Data Encryption

## Giriş

Data Encryption, hassas verilerin güvenliğini sağlamak için kullanılan temel kriptografi teknikleridir. Mid-level geliştiriciler için encryption implementasyonunu anlamak, güvenli veri saklama ve iletim sistemleri tasarlamada kritiktir. Bu dosya, symmetric/asymmetric encryption, hashing, key management ve best practice'leri kapsar.

## Encryption Types

### 1. Symmetric Encryption
Aynı key ile encryption ve decryption yapılan yöntem.

```csharp
public class SymmetricEncryptionService
{
    private readonly ILogger<SymmetricEncryptionService> _logger;
    private readonly IConfiguration _configuration;
    
    public SymmetricEncryptionService(
        ILogger<SymmetricEncryptionService> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }
    
    public async Task<EncryptionResult> EncryptAsync(string plainText, string key = null)
    {
        try
        {
            var encryptionKey = key ?? _configuration["Encryption:SymmetricKey"];
            if (string.IsNullOrEmpty(encryptionKey))
            {
                throw new ArgumentException("Encryption key is required");
            }
            
            using var aes = Aes.Create();
            aes.Key = DeriveKey(encryptionKey, aes.KeySize / 8);
            aes.GenerateIV();
            
            using var encryptor = aes.CreateEncryptor();
            using var msEncrypt = new MemoryStream();
            using var csEncrypt = new CryptoStream(msEncrypt, encryptor, CryptoStreamMode.Write);
            using var swEncrypt = new StreamWriter(csEncrypt);
            
            await swEncrypt.WriteAsync(plainText);
            await swEncrypt.FlushAsync();
            csEncrypt.FlushFinalBlock();
            
            var encryptedData = msEncrypt.ToArray();
            var result = new EncryptionResult
            {
                EncryptedData = Convert.ToBase64String(encryptedData),
                IV = Convert.ToBase64String(aes.IV),
                KeyId = GenerateKeyId(encryptionKey)
            };
            
            _logger.LogDebug("Successfully encrypted data with key ID {KeyId}", result.KeyId);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error encrypting data");
            throw new EncryptionException("Failed to encrypt data", ex);
        }
    }
    
    public async Task<string> DecryptAsync(string encryptedData, string iv, string key = null)
    {
        try
        {
            var encryptionKey = key ?? _configuration["Encryption:SymmetricKey"];
            if (string.IsNullOrEmpty(encryptionKey))
            {
                throw new ArgumentException("Encryption key is required");
            }
            
            using var aes = Aes.Create();
            aes.Key = DeriveKey(encryptionKey, aes.KeySize / 8);
            aes.IV = Convert.FromBase64String(iv);
            
            using var decryptor = aes.CreateDecryptor();
            using var msDecrypt = new MemoryStream(Convert.FromBase64String(encryptedData));
            using var csDecrypt = new CryptoStream(msDecrypt, decryptor, CryptoStreamMode.Read);
            using var srDecrypt = new StreamReader(csDecrypt);
            
            var decryptedText = await srDecrypt.ReadToEndAsync();
            
            _logger.LogDebug("Successfully decrypted data");
            return decryptedText;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error decrypting data");
            throw new EncryptionException("Failed to decrypt data", ex);
        }
    }
    
    public async Task<EncryptionResult> EncryptFileAsync(IFormFile file, string key = null)
    {
        try
        {
            var encryptionKey = key ?? _configuration["Encryption:SymmetricKey"];
            if (string.IsNullOrEmpty(encryptionKey))
            {
                throw new ArgumentException("Encryption key is required");
            }
            
            using var aes = Aes.Create();
            aes.Key = DeriveKey(encryptionKey, aes.KeySize / 8);
            aes.GenerateIV();
            
            using var encryptor = aes.CreateEncryptor();
            using var msEncrypt = new MemoryStream();
            using var csEncrypt = new CryptoStream(msEncrypt, encryptor, CryptoStreamMode.Write);
            
            await file.CopyToAsync(csEncrypt);
            csEncrypt.FlushFinalBlock();
            
            var encryptedData = msEncrypt.ToArray();
            var result = new EncryptionResult
            {
                EncryptedData = Convert.ToBase64String(encryptedData),
                IV = Convert.ToBase64String(aes.IV),
                KeyId = GenerateKeyId(encryptionKey),
                FileName = file.FileName,
                ContentType = file.ContentType
            };
            
            _logger.LogDebug("Successfully encrypted file {FileName} with key ID {KeyId}", 
                file.FileName, result.KeyId);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error encrypting file {FileName}", file.FileName);
            throw new EncryptionException($"Failed to encrypt file {file.FileName}", ex);
        }
    }
    
    private byte[] DeriveKey(string password, int keySize)
    {
        using var deriveBytes = new Rfc2898DeriveBytes(password, new byte[16], 10000, HashAlgorithmName.SHA256);
        return deriveBytes.GetBytes(keySize);
    }
    
    private string GenerateKeyId(string key)
    {
        using var sha256 = SHA256.Create();
        var hash = sha256.ComputeHash(Encoding.UTF8.GetBytes(key));
        return Convert.ToBase64String(hash).Substring(0, 8);
    }
}

public class EncryptionResult
{
    public string EncryptedData { get; set; }
    public string IV { get; set; }
    public string KeyId { get; set; }
    public string FileName { get; set; }
    public string ContentType { get; set; }
}
```

### 2. Asymmetric Encryption
Public/private key pair kullanarak encryption yapılan yöntem.

```csharp
public class AsymmetricEncryptionService
{
    private readonly ILogger<AsymmetricEncryptionService> _logger;
    private readonly IConfiguration _configuration;
    private readonly IKeyVaultService _keyVaultService;
    
    public AsymmetricEncryptionService(
        ILogger<AsymmetricEncryptionService> logger,
        IConfiguration configuration,
        IKeyVaultService keyVaultService)
    {
        _logger = logger;
        _configuration = configuration;
        _keyVaultService = keyVaultService;
    }
    
    public async Task<AsymmetricEncryptionResult> EncryptAsync(string plainText, string keyName = null)
    {
        try
        {
            var keyNameToUse = keyName ?? _configuration["Encryption:AsymmetricKeyName"];
            if (string.IsNullOrEmpty(keyNameToUse))
            {
                throw new ArgumentException("Asymmetric key name is required");
            }
            
            // Get public key from key vault
            var publicKey = await _keyVaultService.GetPublicKeyAsync(keyNameToUse);
            if (publicKey == null)
            {
                throw new ArgumentException($"Public key not found for key name: {keyNameToUse}");
            }
            
            using var rsa = RSA.Create();
            rsa.ImportRSAPublicKey(publicKey, out _);
            
            // Encrypt data in chunks (RSA has size limitations)
            var chunkSize = rsa.KeySize / 8 - 42; // Padding overhead
            var plainTextBytes = Encoding.UTF8.GetBytes(plainText);
            var encryptedChunks = new List<byte[]>();
            
            for (int i = 0; i < plainTextBytes.Length; i += chunkSize)
            {
                var chunk = new byte[Math.Min(chunkSize, plainTextBytes.Length - i)];
                Array.Copy(plainTextBytes, i, chunk, 0, chunk.Length);
                
                var encryptedChunk = rsa.Encrypt(chunk, RSAEncryptionPadding.OaepSHA256);
                encryptedChunks.Add(encryptedChunk);
            }
            
            var result = new AsymmetricEncryptionResult
            {
                EncryptedData = Convert.ToBase64String(encryptedChunks.SelectMany(c => c).ToArray()),
                KeyName = keyNameToUse,
                ChunkCount = encryptedChunks.Count,
                Algorithm = "RSA-OAEP-SHA256"
            };
            
            _logger.LogDebug("Successfully encrypted data with asymmetric key {KeyName}", keyNameToUse);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error encrypting data with asymmetric encryption");
            throw new EncryptionException("Failed to encrypt data with asymmetric encryption", ex);
        }
    }
    
    public async Task<string> DecryptAsync(string encryptedData, string keyName = null)
    {
        try
        {
            var keyNameToUse = keyName ?? _configuration["Encryption:AsymmetricKeyName"];
            if (string.IsNullOrEmpty(keyNameToUse))
            {
                throw new ArgumentException("Asymmetric key name is required");
            }
            
            // Get private key from key vault
            var privateKey = await _keyVaultService.GetPrivateKeyAsync(keyNameToUse);
            if (privateKey == null)
            {
                throw new ArgumentException($"Private key not found for key name: {keyNameToUse}");
            }
            
            using var rsa = RSA.Create();
            rsa.ImportRSAPrivateKey(privateKey, out _);
            
            var encryptedBytes = Convert.FromBase64String(encryptedData);
            var chunkSize = rsa.KeySize / 8;
            var decryptedChunks = new List<byte[]>();
            
            for (int i = 0; i < encryptedBytes.Length; i += chunkSize)
            {
                var chunk = new byte[Math.Min(chunkSize, encryptedBytes.Length - i)];
                Array.Copy(encryptedBytes, i, chunk, 0, chunk.Length);
                
                var decryptedChunk = rsa.Decrypt(chunk, RSAEncryptionPadding.OaepSHA256);
                decryptedChunks.Add(decryptedChunk);
            }
            
            var decryptedBytes = decryptedChunks.SelectMany(c => c).ToArray();
            var decryptedText = Encoding.UTF8.GetString(decryptedBytes);
            
            _logger.LogDebug("Successfully decrypted data with asymmetric key {KeyName}", keyNameToUse);
            return decryptedText;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error decrypting data with asymmetric encryption");
            throw new EncryptionException("Failed to decrypt data with asymmetric encryption", ex);
        }
    }
    
    public async Task<string> SignDataAsync(string data, string keyName = null)
    {
        try
        {
            var keyNameToUse = keyName ?? _configuration["Encryption:AsymmetricKeyName"];
            if (string.IsNullOrEmpty(keyNameToUse))
            {
                throw new ArgumentException("Asymmetric key name is required");
            }
            
            // Get private key from key vault
            var privateKey = await _keyVaultService.GetPrivateKeyAsync(keyNameToUse);
            if (privateKey == null)
            {
                throw new ArgumentException($"Private key not found for key name: {keyNameToUse}");
            }
            
            using var rsa = RSA.Create();
            rsa.ImportRSAPrivateKey(privateKey, out _);
            
            var dataBytes = Encoding.UTF8.GetBytes(data);
            var signature = rsa.SignData(dataBytes, HashAlgorithmName.SHA256, RSASignaturePadding.Pss);
            
            _logger.LogDebug("Successfully signed data with asymmetric key {KeyName}", keyNameToUse);
            return Convert.ToBase64String(signature);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error signing data");
            throw new EncryptionException("Failed to sign data", ex);
        }
    }
    
    public async Task<bool> VerifySignatureAsync(string data, string signature, string keyName = null)
    {
        try
        {
            var keyNameToUse = keyName ?? _configuration["Encryption:AsymmetricKeyName"];
            if (string.IsNullOrEmpty(keyNameToUse))
            {
                throw new ArgumentException("Asymmetric key name is required");
            }
            
            // Get public key from key vault
            var publicKey = await _keyVaultService.GetPublicKeyAsync(keyNameToUse);
            if (publicKey == null)
            {
                throw new ArgumentException($"Public key not found for key name: {keyNameToUse}");
            }
            
            using var rsa = RSA.Create();
            rsa.ImportRSAPublicKey(publicKey, out _);
            
            var dataBytes = Encoding.UTF8.GetBytes(data);
            var signatureBytes = Convert.FromBase64String(signature);
            
            var isValid = rsa.VerifyData(dataBytes, signatureBytes, HashAlgorithmName.SHA256, RSASignaturePadding.Pss);
            
            _logger.LogDebug("Signature verification result: {IsValid} for key {KeyName}", isValid, keyNameToUse);
            return isValid;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error verifying signature");
            return false;
        }
    }
}

public class AsymmetricEncryptionResult
{
    public string EncryptedData { get; set; }
    public string KeyName { get; set; }
    public int ChunkCount { get; set; }
    public string Algorithm { get; set; }
}
```

### 3. Hashing and Salting
Veri bütünlüğü ve güvenliği için hashing algoritmaları.

```csharp
public class HashingService
{
    private readonly ILogger<HashingService> _logger;
    private readonly IConfiguration _configuration;
    
    public HashingService(
        ILogger<HashingService> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }
    
    public async Task<HashResult> HashPasswordAsync(string password)
    {
        try
        {
            // Generate random salt
            var salt = new byte[32];
            using var rng = new RNGCryptoServiceProvider();
            rng.GetBytes(salt);
            
            // Hash password with salt
            using var pbkdf2 = new Rfc2898DeriveBytes(
                password, 
                salt, 
                100000, // Iteration count
                HashAlgorithmName.SHA256);
            
            var hash = pbkdf2.GetBytes(32);
            
            var result = new HashResult
            {
                Hash = Convert.ToBase64String(hash),
                Salt = Convert.ToBase64String(salt),
                Iterations = 100000,
                Algorithm = "PBKDF2-SHA256"
            };
            
            _logger.LogDebug("Successfully hashed password");
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error hashing password");
            throw new HashingException("Failed to hash password", ex);
        }
    }
    
    public async Task<bool> VerifyPasswordAsync(string password, string storedHash, string storedSalt, int iterations)
    {
        try
        {
            var salt = Convert.FromBase64String(storedSalt);
            
            using var pbkdf2 = new Rfc2898DeriveBytes(
                password, 
                salt, 
                iterations, 
                HashAlgorithmName.SHA256);
            
            var hash = pbkdf2.GetBytes(32);
            var computedHash = Convert.ToBase64String(hash);
            
            var isValid = storedHash == computedHash;
            
            _logger.LogDebug("Password verification result: {IsValid}", isValid);
            return isValid;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error verifying password");
            return false;
        }
    }
    
    public async Task<string> HashFileAsync(IFormFile file)
    {
        try
        {
            using var sha256 = SHA256.Create();
            using var stream = file.OpenReadStream();
            
            var hash = await sha256.ComputeHashAsync(stream);
            var hashString = Convert.ToBase64String(hash);
            
            _logger.LogDebug("Successfully hashed file {FileName}", file.FileName);
            return hashString;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error hashing file {FileName}", file.FileName);
            throw new HashingException($"Failed to hash file {file.FileName}", ex);
        }
    }
    
    public async Task<string> HashDataAsync(string data)
    {
        try
        {
            using var sha256 = SHA256.Create();
            var dataBytes = Encoding.UTF8.GetBytes(data);
            var hash = sha256.ComputeHash(dataBytes);
            var hashString = Convert.ToBase64String(hash);
            
            _logger.LogDebug("Successfully hashed data");
            return hashString;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error hashing data");
            throw new HashingException("Failed to hash data", ex);
        }
    }
    
    public async Task<bool> VerifyFileIntegrityAsync(IFormFile file, string expectedHash)
    {
        try
        {
            var actualHash = await HashFileAsync(file);
            var isValid = actualHash == expectedHash;
            
            _logger.LogDebug("File integrity verification result: {IsValid} for {FileName}", isValid, file.FileName);
            return isValid;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error verifying file integrity for {FileName}", file.FileName);
            return false;
        }
    }
}

public class HashResult
{
    public string Hash { get; set; }
    public string Salt { get; set; }
    public int Iterations { get; set; }
    public string Algorithm { get; set; }
}
```

## Key Management

### 1. Key Vault Service
Encryption key'lerini güvenli şekilde yöneten servis.

```csharp
public interface IKeyVaultService
{
    Task<byte[]> GetPublicKeyAsync(string keyName);
    Task<byte[]> GetPrivateKeyAsync(string keyName);
    Task<string> GetSecretAsync(string secretName);
    Task<bool> StoreSecretAsync(string secretName, string secretValue);
    Task<bool> RotateKeyAsync(string keyName);
}

public class AzureKeyVaultService : IKeyVaultService
{
    private readonly ILogger<AzureKeyVaultService> _logger;
    private readonly IConfiguration _configuration;
    private readonly SecretClient _secretClient;
    private readonly KeyClient _keyClient;
    
    public AzureKeyVaultService(
        ILogger<AzureKeyVaultService> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
        
        var keyVaultUrl = configuration["Azure:KeyVault:Url"];
        var credential = new DefaultAzureCredential();
        
        _secretClient = new SecretClient(new Uri(keyVaultUrl), credential);
        _keyClient = new KeyClient(new Uri(keyVaultUrl), credential);
    }
    
    public async Task<byte[]> GetPublicKeyAsync(string keyName)
    {
        try
        {
            var key = await _keyClient.GetKeyAsync(keyName);
            var publicKey = key.Value.Key;
            
            if (publicKey is RsaPublicKey rsaKey)
            {
                return rsaKey.N;
            }
            
            _logger.LogWarning("Key {KeyName} is not an RSA public key", keyName);
            return null;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting public key {KeyName}", keyName);
            return null;
        }
    }
    
    public async Task<byte[]> GetPrivateKeyAsync(string keyName)
    {
        try
        {
            var key = await _keyClient.GetKeyAsync(keyName);
            var privateKey = key.Value.Key;
            
            if (privateKey is RsaPrivateKey rsaKey)
            {
                return rsaKey.D;
            }
            
            _logger.LogWarning("Key {KeyName} is not an RSA private key", keyName);
            return null;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting private key {KeyName}", keyName);
            return null;
        }
    }
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        try
        {
            var secret = await _secretClient.GetSecretAsync(secretName);
            return secret.Value.Value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting secret {SecretName}", secretName);
            return null;
        }
    }
    
    public async Task<bool> StoreSecretAsync(string secretName, string secretValue)
    {
        try
        {
            await _secretClient.SetSecretAsync(secretName, secretValue);
            _logger.LogInformation("Successfully stored secret {SecretName}", secretName);
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error storing secret {SecretName}", secretName);
            return false;
        }
    }
    
    public async Task<bool> RotateKeyAsync(string keyName)
    {
        try
        {
            // Create new key version
            var keyOptions = new CreateRsaKeyOptions(keyName)
            {
                KeySize = 2048,
                KeyOperations = { KeyOperation.Encrypt, KeyOperation.Decrypt, KeyOperation.Sign, KeyOperation.Verify }
            };
            
            var newKey = await _keyClient.CreateRsaKeyAsync(keyOptions);
            
            // Update configuration to use new key
            // This would typically involve updating configuration or database
            
            _logger.LogInformation("Successfully rotated key {KeyName}", keyName);
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error rotating key {KeyName}", keyName);
            return false;
        }
    }
}
```

### 2. Key Rotation Service
Encryption key'lerini otomatik olarak rotate eden servis.

```csharp
public class KeyRotationService : BackgroundService
{
    private readonly ILogger<KeyRotationService> _logger;
    private readonly IConfiguration _configuration;
    private readonly IKeyVaultService _keyVaultService;
    private readonly IServiceProvider _serviceProvider;
    
    public KeyRotationService(
        ILogger<KeyRotationService> logger,
        IConfiguration configuration,
        IKeyVaultService keyVaultService,
        IServiceProvider serviceProvider)
    {
        _logger = logger;
        _configuration = configuration;
        _keyVaultService = keyVaultService;
        _serviceProvider = serviceProvider;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await RotateKeysIfNeededAsync();
                
                // Wait for next rotation check (e.g., daily)
                await Task.Delay(TimeSpan.FromHours(24), stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // Service is stopping
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in key rotation service");
                await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
            }
        }
    }
    
    private async Task RotateKeysIfNeededAsync()
    {
        try
        {
            var keyRotationConfig = _configuration.GetSection("KeyRotation").Get<KeyRotationConfig>();
            if (keyRotationConfig == null)
            {
                _logger.LogWarning("Key rotation configuration not found");
                return;
            }
            
            foreach (var keyConfig in keyRotationConfig.Keys)
            {
                if (await ShouldRotateKeyAsync(keyConfig))
                {
                    await RotateKeyAsync(keyConfig);
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error checking key rotation");
        }
    }
    
    private async Task<bool> ShouldRotateKeyAsync(KeyConfig keyConfig)
    {
        try
        {
            // Check key age
            var keyCreationDate = await GetKeyCreationDateAsync(keyConfig.Name);
            if (keyCreationDate.HasValue)
            {
                var keyAge = DateTime.UtcNow - keyCreationDate.Value;
                if (keyAge.TotalDays >= keyConfig.MaxAgeDays)
                {
                    _logger.LogInformation("Key {KeyName} is {Days} days old, rotation needed", 
                        keyConfig.Name, keyAge.TotalDays);
                    return true;
                }
            }
            
            // Check if key is compromised (this would be based on security events)
            if (await IsKeyCompromisedAsync(keyConfig.Name))
            {
                _logger.LogWarning("Key {KeyName} is compromised, immediate rotation needed", keyConfig.Name);
                return true;
            }
            
            return false;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error checking if key {KeyName} should be rotated", keyConfig.Name);
            return false;
        }
    }
    
    private async Task RotateKeyAsync(KeyConfig keyConfig)
    {
        try
        {
            _logger.LogInformation("Starting key rotation for {KeyName}", keyConfig.Name);
            
            // Create new key
            var newKeyCreated = await _keyVaultService.RotateKeyAsync(keyConfig.Name);
            if (!newKeyCreated)
            {
                _logger.LogError("Failed to create new key for {KeyName}", keyConfig.Name);
                return;
            }
            
            // Update configuration
            await UpdateKeyConfigurationAsync(keyConfig.Name);
            
            // Notify stakeholders
            await NotifyKeyRotationAsync(keyConfig.Name);
            
            _logger.LogInformation("Successfully completed key rotation for {KeyName}", keyConfig.Name);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error rotating key {KeyName}", keyConfig.Name);
        }
    }
    
    private async Task<DateTime?> GetKeyCreationDateAsync(string keyName)
    {
        // Implementation to get key creation date from key vault
        // This is a simplified example
        return DateTime.UtcNow.AddDays(-30);
    }
    
    private async Task<bool> IsKeyCompromisedAsync(string keyName)
    {
        // Implementation to check if key is compromised
        // This would typically involve checking security logs, alerts, etc.
        return false;
    }
    
    private async Task UpdateKeyConfigurationAsync(string keyName)
    {
        // Implementation to update configuration with new key
        // This could involve updating database, configuration files, etc.
        await Task.CompletedTask;
    }
    
    private async Task NotifyKeyRotationAsync(string keyName)
    {
        // Implementation to notify stakeholders about key rotation
        // This could involve sending emails, Slack messages, etc.
        await Task.CompletedTask;
    }
}

public class KeyRotationConfig
{
    public List<KeyConfig> Keys { get; set; } = new List<KeyConfig>();
}

public class KeyConfig
{
    public string Name { get; set; }
    public int MaxAgeDays { get; set; } = 90;
    public bool AutoRotation { get; set; } = true;
}
```

## Database Encryption

### 1. Column-Level Encryption
Veritabanında belirli kolonları encrypt eden servis.

```csharp
public class DatabaseEncryptionService
{
    private readonly ILogger<DatabaseEncryptionService> _logger;
    private readonly IConfiguration _configuration;
    private readonly SymmetricEncryptionService _symmetricEncryptionService;
    
    public DatabaseEncryptionService(
        ILogger<DatabaseEncryptionService> logger,
        IConfiguration configuration,
        SymmetricEncryptionService symmetricEncryptionService)
    {
        _logger = logger;
        _configuration = configuration;
        _symmetricEncryptionService = symmetricEncryptionService;
    }
    
    public async Task<string> EncryptColumnValueAsync(string plainText, string columnName)
    {
        try
        {
            if (string.IsNullOrEmpty(plainText))
            {
                return plainText;
            }
            
            var encryptionResult = await _symmetricEncryptionService.EncryptAsync(plainText);
            
            // Store IV with encrypted data for decryption
            var encryptedValue = $"{encryptionResult.IV}:{encryptionResult.EncryptedData}";
            
            _logger.LogDebug("Successfully encrypted column {ColumnName}", columnName);
            return encryptedValue;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error encrypting column {ColumnName}", columnName);
            throw new DatabaseEncryptionException($"Failed to encrypt column {columnName}", ex);
        }
    }
    
    public async Task<string> DecryptColumnValueAsync(string encryptedValue, string columnName)
    {
        try
        {
            if (string.IsNullOrEmpty(encryptedValue) || !encryptedValue.Contains(":"))
            {
                return encryptedValue; // Not encrypted or invalid format
            }
            
            var parts = encryptedValue.Split(':', 2);
            if (parts.Length != 2)
            {
                _logger.LogWarning("Invalid encrypted value format for column {ColumnName}", columnName);
                return encryptedValue;
            }
            
            var iv = parts[0];
            var encryptedData = parts[1];
            
            var decryptedValue = await _symmetricEncryptionService.DecryptAsync(encryptedData, iv);
            
            _logger.LogDebug("Successfully decrypted column {ColumnName}", columnName);
            return decryptedValue;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error decrypting column {ColumnName}", columnName);
            throw new DatabaseEncryptionException($"Failed to decrypt column {columnName}", ex);
        }
    }
    
    public async Task<Dictionary<string, object>> EncryptEntityAsync<T>(T entity) where T : class
    {
        try
        {
            var encryptedEntity = new Dictionary<string, object>();
            var entityType = typeof(T);
            var encryptedProperties = GetEncryptedProperties(entityType);
            
            foreach (var property in entityType.GetProperties())
            {
                var value = property.GetValue(entity);
                
                if (encryptedProperties.Contains(property.Name) && value != null)
                {
                    var encryptedValue = await EncryptColumnValueAsync(value.ToString(), property.Name);
                    encryptedEntity[property.Name] = encryptedValue;
                }
                else
                {
                    encryptedEntity[property.Name] = value;
                }
            }
            
            _logger.LogDebug("Successfully encrypted entity of type {EntityType}", entityType.Name);
            return encryptedEntity;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error encrypting entity of type {EntityType}", typeof(T).Name);
            throw new DatabaseEncryptionException($"Failed to encrypt entity of type {typeof(T).Name}", ex);
        }
    }
    
    public async Task<T> DecryptEntityAsync<T>(Dictionary<string, object> encryptedEntity) where T : class, new()
    {
        try
        {
            var entity = new T();
            var entityType = typeof(T);
            var encryptedProperties = GetEncryptedProperties(entityType);
            
            foreach (var kvp in encryptedEntity)
            {
                var property = entityType.GetProperty(kvp.Key);
                if (property != null && property.CanWrite)
                {
                    var value = kvp.Value;
                    
                    if (encryptedProperties.Contains(kvp.Key) && value != null)
                    {
                        var decryptedValue = await DecryptColumnValueAsync(value.ToString(), kvp.Key);
                        property.SetValue(entity, Convert.ChangeType(decryptedValue, property.PropertyType));
                    }
                    else
                    {
                        property.SetValue(entity, Convert.ChangeType(value, property.PropertyType));
                    }
                }
            }
            
            _logger.LogDebug("Successfully decrypted entity of type {EntityType}", entityType.Name);
            return entity;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error decrypting entity of type {EntityType}", typeof(T).Name);
            throw new DatabaseEncryptionException($"Failed to decrypt entity of type {typeof(T).Name}", ex);
        }
    }
    
    private HashSet<string> GetEncryptedProperties(Type entityType)
    {
        var encryptedProperties = new HashSet<string>();
        
        foreach (var property in entityType.GetProperties())
        {
            var encryptedAttribute = property.GetCustomAttribute<EncryptedAttribute>();
            if (encryptedAttribute != null)
            {
                encryptedProperties.Add(property.Name);
            }
        }
        
        return encryptedProperties;
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class EncryptedAttribute : Attribute
{
    public string EncryptionKey { get; set; }
}

public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    
    [Encrypted]
    public string Email { get; set; }
    
    [Encrypted]
    public string PhoneNumber { get; set; }
    
    public DateTime CreatedAt { get; set; }
}
```

## Mülakat Soruları

### Temel Sorular

1. **Symmetric vs Asymmetric encryption arasındaki fark nedir?**
   - **Cevap**: Symmetric aynı key kullanır, hızlı ama key distribution zor. Asymmetric public/private key pair kullanır, yavaş ama key distribution kolay.

2. **Hashing vs Encryption arasındaki fark nedir?**
   - **Cevap**: Hashing one-way, veri bütünlüğü için. Encryption reversible, veri gizliliği için.

3. **Salt nedir ve neden kullanılır?**
   - **Cevap**: Random data eklenir, rainbow table attacks'i önler, aynı password'lar için farklı hash'ler üretir.

4. **Key rotation neden önemlidir?**
   - **Cevap**: Compromised key'lerin etkisini sınırlar, security posture'ı iyileştirir, compliance gereksinimlerini karşılar.

5. **Column-level encryption ne zaman kullanılır?**
   - **Cevap**: Hassas veriler (SSN, credit card, health data) için, compliance gereksinimlerinde, selective data protection için.

### Teknik Sorular

1. **RSA encryption'da chunk size nasıl hesaplanır?**
   - **Cevap**: Key size / 8 - padding overhead (42 bytes for OAEP-SHA256).

2. **PBKDF2'de iteration count neden önemlidir?**
   - **Cevap**: Brute force attacks'i zorlaştırır, computational cost artırır, security level belirler.

3. **Database encryption'da IV nasıl saklanır?**
   - **Cevap**: Encrypted data ile birlikte (IV:encrypted_data format), ayrı kolonda, metadata olarak.

4. **Key vault'ta key rotation nasıl implement edilir?**
   - **Cevap**: New key version oluştur, configuration update, gradual migration, old key cleanup.

5. **Encryption performance nasıl optimize edilir?**
   - **Cevap**: Hardware acceleration, async operations, caching, selective encryption, key size optimization.

## Best Practices

1. **Key Management**
   - Key vault kullanın
   - Regular key rotation yapın
   - Key access logging implement edin
   - Backup ve recovery planı hazırlayın

2. **Encryption Implementation**
   - Strong algorithms kullanın (AES-256, RSA-2048)
   - Secure random number generation kullanın
   - IV'leri unique yapın
   - Proper padding schemes kullanın

3. **Performance Optimization**
   - Hardware acceleration kullanın
   - Async operations implement edin
   - Selective encryption yapın
   - Caching strategies kullanın

4. **Security Monitoring**
   - Encryption operations log edin
   - Key usage metrics toplayın
   - Security alerts kurun
   - Regular security audits yapın

5. **Compliance & Standards**
   - Industry standards takip edin (FIPS, NIST)
   - Regular security assessments yapın
   - Documentation güncelleyin
   - Training programs kurun

## Kaynaklar

- [Microsoft Encryption Documentation](https://docs.microsoft.com/en-us/dotnet/standard/security/cryptography-model)
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [NIST Cryptographic Standards](https://www.nist.gov/cryptography)
- [Azure Key Vault Documentation](https://docs.microsoft.com/en-us/azure/key-vault/)
- [Cryptography Best Practices](https://cryptography.io/en/latest/)
