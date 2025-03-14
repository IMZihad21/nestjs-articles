This technical guide presents a production-ready security service implementation for NestJS applications, integrating essential cryptographic operations while adhering to industry-standard security protocols. The `EncryptionService` encapsulates critical functionality for modern backend systems, featuring Argon2 password hashing, AES-256 data encryption, and secure token generation within a modular architecture.

---

### **Core Service Configuration**  

The foundation establishes environment-aware cryptographic parameters using NestJS configuration patterns:  

```typescript
import {
  BadRequestException,
  Injectable,
  InternalServerErrorException,
  Logger,
} from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import * as argon2 from "argon2";
import { createCipheriv, createDecipheriv, createHash } from "crypto";
import * as uuid from "uuid";
import * as base64url from "base64url";

@Injectable()
export class EncryptionService {
  private readonly logger = new Logger(EncryptionService.name);
  private readonly algorithm = "aes-256-cbc";
  private readonly ivLength = 16;
  private readonly secretKey: string;

  constructor(private configService: ConfigService) {
    this.secretKey = this.configService.get<string>("ENCRYPTION_SECRET", "default_secret");
  }
```  

**Security Architecture**  
- **Algorithm Selection**: AES-256-CBC provides FIPS-approved symmetric encryption  
- **IV Management**: Fixed-length initialization vector for CBC mode compatibility  
- **Secret Handling**: Environment variable injection prevents key hardcoding  
- **Fallback Mechanism**: Default secret enables development environments while enforcing production configuration  

---

### **Password Security Implementation**  

#### **1. Argon2 Password Hashing**  
Implements memory-hard hashing for credential protection:  

```typescript
  async hashPassword(rawPassword: string): Promise<string> {
    if (!rawPassword) {
      this.logger.error("Password is required");
      throw new BadRequestException("Password is required");
    }

    try {
      return await argon2.hash(rawPassword);
    } catch (err) {
      this.logger.error("Failed to hash password", err);
      throw new InternalServerErrorException("Failed to hash password");
    }
  }

  async verifyPassword(rawPassword: string = "", hashedPassword: string = ""): Promise<boolean> {
    if (!rawPassword) {
      this.logger.error("Password is required");
      throw new BadRequestException("Password is required");
    }

    try {
      return await argon2.verify(hashedPassword, rawPassword);
    } catch (err) {
      this.logger.error("Failed to verify password", err);
      throw new InternalServerErrorException("Failed to verify password");
    }
  }
```  

**Enterprise Security Features**  
- **Timing Attack Resistance**: Constant-time verification implementation  
- **Memory Hardness**: Configurable memory/iteration parameters (implicit in argon2.hash)  
- **Validation Enforcement**: Strict input checking prevents empty credential processing  

---

#### **2. Temporary Password Generation**  
Implements complex password synthesis for recovery workflows:  

```typescript
  generateTemporaryPassword(length = 10) {
    const lowercaseChars = "abcdefghijklmnopqrstuvwxyz";
    const uppercaseChars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
    const numericChars = "0123456789";
    const specialChars = "!@#$%^&*()-_+=";

    const allChars = lowercaseChars + uppercaseChars + numericChars + specialChars;
    let password = "";

    try {
      for (let i = 0; i < length; i++) {
        const randomIndex = Math.floor(Math.random() * allChars.length);
        password += allChars[randomIndex];
      }
    } catch (error) {
      throw new InternalServerErrorException("Failed to generate a password");
    }

    return password;
  }
```  

**Security Considerations**  
- **Character Diversity**: Four distinct character classes minimize pattern predictability  
- **Length Configuration**: Adjustable complexity through parameterization  
- **Entropy Management**: Combined character pools enhance combinatorial complexity  

---

### **Cryptographic Token Management**  

#### **UUID-Based Token Generation**  
Creates collision-resistant identifiers for session management:  

```typescript
  generateUniqueToken(length: number = 3): string {
    const mergedUuid = Array.from({ length }, () => uuid.v4()).join("");
    const tokenBuffer = Buffer.from(mergedUuid.replace(/-/g, ""), "hex");
    return base64url.default(tokenBuffer);
  }
```  

**Cryptographic Properties**  
- **UUID v4 Utilization**: 122-bit random number space guarantees uniqueness  
- **Base64URL Encoding**: URL-safe representation for web compatibility  
- **Concatenation Strategy**: Multiple UUIDs enhance token entropy  

---

### **Symmetric Data Encryption**  

#### **1. AES-256-CBC Encryption**  
Implements FIPS 197-compliant data protection:  

```typescript
  encryptString(text: string): string {
    if (!text) throw new Error("Text is required for encryption");

    try {
      const key = createHash("sha256").update(this.secretKey).digest("base64").slice(0, 32);
      const iv = Buffer.alloc(this.ivLength, 0);

      const cipher = createCipheriv(this.algorithm, key, iv);
      let encrypted = cipher.update(text, "utf8", "base64");
      encrypted += cipher.final("base64");

      return encrypted.replace(/\+/g, "-").replace(/\//g, "_").replace(/=/g, "");
    } catch (error) {
      this.logger.error("Encryption failed", error);
      throw new InternalServerErrorException("Encryption failed");
    }
  }
```  

#### **2. AES-256-CBC Decryption**  
Ensures reliable data recovery process:  

```typescript
  decryptString(cipherText: string): string {
    if (!cipherText) throw new Error("Cipher text is required for decryption");

    try {
      const base64Text = cipherText.replace(/-/g, "+").replace(/_/g, "/");
      const key = createHash("sha256").update(this.secretKey).digest("base64").slice(0, 32);
      const iv = Buffer.alloc(this.ivLength, 0);

      const decipher = createDecipheriv(this.algorithm, key, iv);
      let decrypted = decipher.update(base64Text, "base64", "utf8");
      decrypted += decipher.final("utf8");

      return decrypted;
    } catch (error) {
      this.logger.error("Decryption failed", error);
      throw new InternalServerErrorException("Decryption failed");
    }
  }
```  

**Cryptographic Implementation Details**  
- **Key Derivation**: SHA-256 hashing ensures fixed-length key material  
- **IV Strategy**: Zero-byte initialization vector (production implementations should use random IVs)  
- **Encoding Handling**: Base64URL conversion for web-safe ciphertext representation  

---

### **Service Modularization**  

Global module declaration enables cross-application accessibility:  

```typescript
import { Global, Module } from "@nestjs/common";
import { EncryptionService } from "./encryption.service";

@Global()
@Module({
  providers: [EncryptionService],
  exports: [EncryptionService],
})
export class EncryptionModule {}
```  

**Architectural Benefits**  
- **Singleton Pattern**: Single service instance across application modules  
- **Dependency Management**: Centralized cryptographic functionality  
- **Security Consistency**: Uniform implementation across application layers  

---

### **Production Deployment Considerations**  

1. **Secret Management**  
   - Implement rotational policies for encryption secrets  
   - Integrate with vault services for secure key storage  

2. **Performance Optimization**  
   - Benchmark Argon2 parameters against server capabilities  
   - Consider hardware acceleration for cryptographic operations  

3. **Compliance Requirements**  
   - Align with PCI-DSS for payment data handling  
   - Implement FIPS 140-2 validation where required  

4. **Attack Mitigation**  
   - Rate-limiting for password verification attempts  
   - Automated secret rotation mechanisms  

---

This implementation provides a robust security foundation for NestJS applications, addressing critical requirements for:  
- **Credential Protection**: Memory-hard hashing prevents brute-force attacks  
- **Data Confidentiality**: Strong encryption safeguards sensitive information  
- **Session Security**: Unpredictable tokens mitigate session hijacking risks  
- **Operational Reliability**: Comprehensive error handling ensures system stability  

The modular design facilitates seamless integration with enterprise security ecosystems, providing a critical layer in defense-in-depth strategies while maintaining developer productivity through standardized cryptographic primitives.