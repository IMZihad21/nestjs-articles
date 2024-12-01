Security is a top priority when building backend services, especially in applications dealing with sensitive data. In this guide, we'll build an EncryptionService in NestJS that covers a broad range of security features, including:

- **Password hashing and verification** using Argon2, known for its high security and resistance to attacks.
- **Unique token generation** for actions that require unique identifiers or session handling.
- **AES encryption and decryption** for secure data storage or transmission.

This multi-featured approach provides a robust solution to common security needs in a backend service.

## Setting Up the EncryptionService

First, we’ll configure the service with environment-based encryption parameters. By utilizing NestJS's `ConfigService`, we can manage sensitive information, such as encryption keys, more securely.

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

In this configuration:

- **Algorithm and IV Length**: Set to aes-256-cbc and an IV length of 16 bytes, respectively, for strong encryption.
- **Secret Key**: Loaded from environment variables, offering flexibility and enhancing security.

## Secure Password Hashing with Argon2

Argon2 is a memory-hard algorithm designed to be secure against brute-force attacks, making it ideal for password storage. We’ll implement methods to hash passwords and verify hashed passwords with raw inputs.

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

- **hashPassword**: Creates a secure hash from a raw password using Argon2, with error handling for unexpected issues.
- **verifyPassword**: Compares a raw password to a hashed password, ensuring only authorized access is allowed.

## Temporary Password Generation
The `generateTemporaryPassword` function in the `EncryptionService` provides a way to create secure, random passwords. This is particularly useful in scenarios where users need a temporary password or a randomly generated password as part of the account recovery process or for initial account setup.

Here's a breakdown of how this function works:

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

This method is flexible and allows you to specify different password lengths by passing a length parameter, defaulting to 10 characters if no length is provided.

## Unique Token Generation

In cases where a unique identifier is needed, such as session IDs or verification tokens, generating a unique, hard-to-guess string is critical. Here, we use UUID and base64 encoding for this purpose.

```typescript
  generateUniqueToken(length: number = 3): string {
    const mergedUuid = Array.from({ length }, () => uuid.v4()).join("");
    const tokenBuffer = Buffer.from(mergedUuid.replace(/-/g, ""), "hex");
    return base64url.default(tokenBuffer);
  }
```

This method generates a unique token using multiple UUIDs for greater uniqueness, making it suitable for scenarios requiring temporary identifiers.

## AES Encryption and Decryption for Sensitive Data

AES encryption is widely used for securing sensitive data. Below, we create methods for encrypting and decrypting strings using AES-256 with a CBC mode.

### Encrypting Data

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

### Decrypting Data

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

The encryption and decryption methods use a secure key derived from the secret key. The encrypted data is base64-encoded for safer storage and transmission.


## Setting Up the `EncryptionModule`

To make `EncryptionService` available across the application, we encapsulate it within a globally accessible module: `EncryptionModule`.

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

## Conclusion

This `EncryptionService` implementation provides a highly secure way to handle encryption, hashing, and token generation in a NestJS application. It uses best-in-class algorithms such as Argon2 for hashing and AES-256 for encryption, making it versatile and secure for any backend project.

With this setup, you’ll have a robust foundation for safeguarding sensitive information, enabling more secure application development.






