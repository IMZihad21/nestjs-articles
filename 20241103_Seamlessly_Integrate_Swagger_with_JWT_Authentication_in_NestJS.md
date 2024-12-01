Enhancing your NestJS API with robust documentation and JWT authentication is essential for a smooth developer experience. In this article, we'll walk you through integrating Swagger UI into your NestJS application, utilizing dummy data, and customizing the UI with external assets.

## Step 1: Install Required Packages

Start by installing the necessary packages for Swagger support:

```bash
npm install @nestjs/swagger
```

## Step 2: Create Swagger Configuration

Create a file (`swagger.config.ts`), configure Swagger using the following code snippet with dummy data for demonstration:

```typescript
import { INestApplication } from "@nestjs/common";
import {
  DocumentBuilder,
  SwaggerCustomOptions,
  SwaggerModule,
} from "@nestjs/swagger";

const documentConfig = new DocumentBuilder()
  .setTitle("Dummy API")  // API Title
  .setVersion("1.0.0")  // API Version
  .addBearerAuth(  // Adds JWT Bearer authentication
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
  .addSecurityRequirements("Bearer")  // Security requirement
  .build();

const swaggerUiOptions: SwaggerCustomOptions = {
  swaggerOptions: {
    persistAuthorization: true,  // Retain authorization
  },
  customSiteTitle: "Dummy API Documentation",  // Swagger UI title
  customJs: [  // External JS files
    "https://unpkg.com/swagger-ui-dist@4.5.0/swagger-ui-bundle.js",
    "https://unpkg.com/swagger-ui-dist@4.5.0/swagger-ui-standalone-preset.js",
  ],
  customCssUrl: [  // External CSS files
    "https://unpkg.com/swagger-ui-dist@4.5.0/swagger-ui.css",
  ],
};

// Function to configure Swagger UI
export const configureSwaggerUI = (app: INestApplication<any>) => {
  const document = SwaggerModule.createDocument(app, documentConfig);  // Create Swagger document
  SwaggerModule.setup("swagger", app, document, swaggerUiOptions);  // Setup Swagger UI
};
```

### Explanation

- **Document Configuration**: Set up a Swagger document with a title, version, and authentication.
- **Swagger UI Options**: Configure options, including a custom title and external assets.

## Step 3: Integrate Swagger in Your NestJS Application

Add the Swagger configuration in your `main.ts` file:

```typescript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { configureSwaggerUI } from "./swagger.config";  // Import Swagger configuration

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  configureSwaggerUI(app);  // Configure Swagger
  await app.listen(4000);  // Start the application
}

bootstrap();
```

## Step 4: Run Your Application

Now, run your NestJS application:

```bash
npm run start
```

## Step 5: Access Swagger UI

Open your browser and navigate to:

```
http://localhost:4000/swagger
```

You should see the Swagger UI with your Dummy API documentation!