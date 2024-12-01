Effective error logging is vital for application stability. Integrating Sentry into a NestJS application can significantly improve how we handle logging. This article will guide you through creating a custom logger that sends error logs to Sentry while keeping the standard console logging.

## Step 1: Install Sentry

First, install the Sentry SDK:

```bash
npm install @sentry/node
```

## Step 2: Create the Custom Logger

Create a new file named `sentry.logger.ts`:

```typescript
import { ConsoleLogger } from "@nestjs/common";
import * as Sentry from "@sentry/node";

export class SentryLogger extends ConsoleLogger {
  error(message: any, ...optionalParams: any[]): void {
    const errorMessage = message.toString();
    let stack: string | object = "";
    let logContext = "";

    if (optionalParams.length === 1) {
      logContext = optionalParams[0];
    } else if (optionalParams.length === 2) {
      [stack, logContext] = optionalParams;
    }

    const formattedMessage = logContext
      ? `${logContext}: ${errorMessage}`
      : errorMessage;

    Sentry.withScope((scope) => {
      scope.setExtra("message", errorMessage);
      scope.setExtra("context", logContext);
      scope.setExtra("stack", stack);
      Sentry.captureMessage(formattedMessage, "error");
    });

    super.error(errorMessage, ...optionalParams);
  }

  verbose(message: any, ...optionalParams: any[]): void {
    const verboseMessage = message ? message.toString() : "";
    const logContext = optionalParams.shift() || "";
    const extra = optionalParams;

    const formattedMessage = logContext
      ? `${logContext}: ${verboseMessage}`
      : verboseMessage;

    Sentry.withScope((scope) => {
      scope.setExtra("message", verboseMessage);
      scope.setExtra("context", logContext);
      scope.setExtra("extra", extra);
      Sentry.captureMessage(formattedMessage, "info");
    });

    super.verbose(verboseMessage, ...extra);
  }
}
```

### Explanation

- **Error Handling**: The `error` method formats the error message and sends it to Sentry, attaching extra context.
- **Verbose Logging**: The `verbose` method captures informational messages, providing insights into the application's state.

## Step 3: Create the Sentry Exception Filter

Next, create an exception filter that captures exceptions and sends detailed information to Sentry. Create a file named `sentry-exception.filter.ts`:

```typescript
import { Catch, Provider, type ArgumentsHost } from "@nestjs/common";
import { APP_FILTER, BaseExceptionFilter } from "@nestjs/core";
import * as Sentry from "@sentry/node";

@Catch()
class SentryExceptionFilter extends BaseExceptionFilter {
  catch(exception: any, host: ArgumentsHost) {
    const http = host.switchToHttp();
    const request = http.getRequest();

    Sentry.withScope((scope) => {
      if (request) {
        scope.setTag("url", request?.url);
        scope.setExtra("request", {
          url: request.url,
          method: request.method,
          headers: request.headers,
          params: request.params,
          body: JSON.stringify(request.body ?? {}),
        });

        if (request?.user) {
          scope.setUser({
            id: request?.user?.userId,
            email: request?.user?.userEmail,
            username: request?.user?.userName,
            Role: request?.user?.userRole,
          });
        }

        scope.setTransactionName(Date.now().toString());
        scope.setTag("environment", process.env.NODE_ENV || "development");
        scope.setExtra("timestamp", new Date().toISOString());
      }

      if (exception.response) {
        scope.setExtra("response", exception.response);
        scope.setTag("status_code", exception.response?.statusCode);
      }

      Sentry.captureException(exception);
    });

    super.catch(exception, host);
  }
}

export const SentryExceptionFilterProvider: Provider = {
  provide: APP_FILTER,
  useClass: SentryExceptionFilter,
};
```

### Explanation

- **Contextual Data**: The filter captures the request data, including URL, method, headers, and body, providing valuable context for debugging.
- **User Information**: If a user is authenticated, their details are added to the Sentry scope.
- **Response Data**: The filter captures response information, including the status code, for further insight into the exception.
- **Capture Exception**: Finally, the exception is sent to Sentry for tracking.


## Step 4: Initialize Sentry

In your main application file (e.g., `main.ts`), initialize Sentry with the following configuration:

```typescript
import { Logger } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { NestFactory } from "@nestjs/core";
import { NestExpressApplication } from "@nestjs/platform-express";
import * as Sentry from "@sentry/node";
import { nodeProfilingIntegration } from "@sentry/profiling-node";
import { AppModule } from "./app.module";
import { SentryLogger } from "./utility/logger/sentry.logger";

const logger = new Logger("MyApp");

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  const cfg = app.get(ConfigService);

  Sentry.init({
    dsn: cfg.get<string>("SENTRY_DNS", ""),
    environment: cfg.get<string>("NODE_ENV", "development"),
    tracesSampleRate: 1.0,
    profilesSampleRate: 1.0,
    normalizeDepth: 5,
    integrations: [nodeProfilingIntegration()],
  });

  app.useLogger(new SentryLogger());
  
  await app.listen(3000);
}

bootstrap()
  .then(() => logger.log("Server is running"))
  .catch((err) => logger.error("Bootstrap failed", err));
```

### Explanation of Sentry Initialization

- **DSN**: The Data Source Name (DSN) allows Sentry to identify and route errors to the correct project.
- **Environment**: Setting the environment (e.g., development or production) helps in distinguishing logs.
- **Sample Rates**: Both tracesSampleRate and profilesSampleRate control how much data is sent to Sentry for performance monitoring, set to 100% (1.0) for full data capture.
- **Integrations**: The nodeProfilingIntegration() is included for additional profiling features.

## Step 5: Register the Sentry Exception Filter in AppModule

Finally, you need to ensure that the `SentryExceptionFilterProvider` is included in your `AppModule`. This will allow the filter to catch exceptions globally within your application. Update your `AppModule` as follows:

```typescript
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";
import { ValidationProvider } from "./validation.provider";
import { SentryExceptionFilterProvider } from "./utility/logger/sentry-exception.filter";

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService, ValidationProvider, SentryExceptionFilterProvider],
})
export class AppModule {}
```

### Explanation

- **Providers Array**: The `SentryExceptionFilterProvider` is added to the `providers` array, which registers it as a global exception filter in the NestJS dependency injection system.
- **Global Exception Handling**: This allows the application to use the custom exception handling logic defined in `SentryExceptionFilter`, ensuring that any unhandled exceptions are properly captured and reported to Sentry.

---

Integrating Sentry into your NestJS application not only helps you log errors but also enhances your application's robustness through effective error tracking and monitoring. By following these steps, you can seamlessly implement a custom logger and a comprehensive exception filter that provide rich context for debugging. Start using Sentry today to improve your application's error management!