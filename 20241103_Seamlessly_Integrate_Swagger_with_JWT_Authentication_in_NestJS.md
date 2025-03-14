Establishing thorough API documentation and secure authentication mechanisms represents a critical requirement for modern backend development. This technical guide demonstrates how to integrate Swagger UI documentation with JSON Web Token (JWT) authentication in a NestJS application while maintaining production-grade standards.  

---

### **Step 1: Dependency Installation**  

Begin by installing the official Swagger module for NestJS:  

```bash
npm install @nestjs/swagger
```  

---

### **Step 2: Swagger Configuration Setup**  

Create a dedicated configuration file (`swagger.config.ts`) with the following implementation:  

```typescript
import { INestApplication } from "@nestjs/common";
import {
  DocumentBuilder,
  SwaggerCustomOptions,
  SwaggerModule,
} from "@nestjs/swagger";

const documentConfig = new DocumentBuilder()
  .setTitle("Dummy API")
  .setVersion("1.0.0")
  .addBearerAuth(
    {
      type: "http",
      scheme: "bearer",
      bearerFormat: "JWT",
      name: "Authorization",
      description: "Enter JWT token",
      in: "header",
    },
    "Bearer",
  )
  .addSecurityRequirements("Bearer")
  .build();

const swaggerUiOptions: SwaggerCustomOptions = {
  swaggerOptions: {
    persistAuthorization: true,
  },
  customSiteTitle: "Dummy API Documentation",
  customJs: [
    "https://unpkg.com/swagger-ui-dist@4.5.0/swagger-ui-bundle.js",
    "https://unpkg.com/swagger-ui-dist@4.5.0/swagger-ui-standalone-preset.js",
  ],
  customCssUrl: [
    "https://unpkg.com/swagger-ui-dist@4.5.0/swagger-ui.css",
  ],
};

export const configureSwaggerUI = (app: INestApplication<any>) => {
  const document = SwaggerModule.createDocument(app, documentConfig);
  SwaggerModule.setup("swagger", app, document, swaggerUiOptions);
};
```  

#### **Configuration Breakdown**  
- **Document Builder**: Defines API metadata and implements JWT authentication schema using Bearer tokens  
- **UI Customization**: Configures persistent authorization and integrates external Swagger assets for interface customization  

---

### **Step 3: Application Integration**  

Implement the Swagger configuration in the application entry point (`main.ts`):  

```typescript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { configureSwaggerUI } from "./swagger.config";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  configureSwaggerUI(app);
  await app.listen(4000);
}

bootstrap();
```  

---

### **Step 4: Application Execution**  

Launch the development server using:  

```bash
npm run start
```  

---

### **Step 5: Documentation Access**  

Access the interactive API documentation through your preferred browser at:  

```
http://localhost:4000/swagger
```  

The Swagger interface will display all implemented endpoints with integrated JWT authentication capabilities.  

---

This implementation strategy provides a standardized approach to API documentation while maintaining security best practices. The configuration allows for seamless extension with additional authentication providers or documentation enhancements, serving as a foundation for scalable NestJS application development.