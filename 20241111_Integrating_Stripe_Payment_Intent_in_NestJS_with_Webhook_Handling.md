In this article, we'll walk through how to implement Stripe payment processing in a NestJS application, focusing on creating a payment intent for bookings and handling webhook events from Stripe. We'll cover the controller, service, and the business logic involved in interacting with Stripe's API, including updating the booking status based on Stripe's payment events.

## 1. Setup Nest Application With Express

To properly handle Stripe webhooks in a NestJS application, one key aspect to ensure smooth processing is retaining the raw request body. This is particularly important for webhook signature verification. To prevent NestJS from parsing the body, you must set rawBody: true in the application setup. This ensures the request body remains in its raw form, allowing Stripe's signature verification process to function properly.

```typescript
async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule, {
    rawBody: true,  // Retain raw request body for Stripe webhook verification
  });


    // ...
}
```

## 2. Setting Up the Controller

The controller handles incoming HTTP requests related to payment processing. We define two routes: one for retrieving a payment intent based on a booking ID and another for handling webhook events sent by Stripe.

```typescript
@ApiTags("Payment Receive")
@Controller("PaymentReceive")
export class PaymentReceiveController {
  constructor(private readonly paymentService: PaymentReceiveService) {}

  @Get("GetIntentByBookingId/:DocId")
  @ApiResponse({ status: 200, type: SuccessResponseDto })
  getPaymentIntent(
    @AuthUserId() { userId }: ITokenPayload,
    @Param() { DocId }: DocIdQueryDto,
  ) {
    return this.paymentService.getPaymentIntent(DocId, userId);
  }

  @Post("StripeWebhook")
  @HttpCode(200)
  @IsPublic()
  @ApiExcludeEndpoint()
  async handleStripeWebhook(
    @Req() req: RawBodyRequest<Request>,
    @Headers("stripe-signature") signature: string,
  ): Promise<void> {
    await this.paymentService.handleStripeWebhook(req, signature);
  }
}
```

### Explanation:

- **Get: GetIntentByBookingId/:DocId**: This route fetches the payment intent associated with a booking by its DocId. The AuthUserId decorator ensures that the user is authenticated.
- **Post: StripeWebhook**: This route handles the incoming webhook notifications from Stripe. The `@IsPublic()` decorator allows this route to be accessed without authentication since webhooks are sent by Stripe, not the user. The `@ApiExcludeEndpoint()` decorator hides this endpoint for swagger doc schema. The `@Req() req: RawBodyRequest<Request>` params provides raw request body from stripe webhook.

## 3. The Payment Receive Service

This service handles the business logic of creating and retrieving payment intents from Stripe, as well as processing the events sent via Stripe webhooks. The key logic revolves around managing payment statuses and updating related entities in the database.

```typescript
@Injectable()
export class StripePaymentService {
  private readonly logger: Logger = new Logger(StripePaymentService.name);
  private readonly stripeService: Stripe;

  private readonly stripeWebhookSecret: string;
  private readonly stripePublishableKey: string;

  constructor(
    private readonly configService: ConfigService,
  ) {
    const stripeSecretKey = this.configService.get<string>("STRIPE_SECRET_KEY", "");
    this.stripePublishableKey = this.configService.get<string>("STRIPE_PUBLISHABLE_KEY", "");
    this.stripeWebhookSecret = this.configService.get<string>("STRIPE_WEBHOOK_SECRET", "");

    this.stripeService = new Stripe(stripeSecretKey, {});
  }

  async getPaymentIntent(
    bookingId: string,
    auditUserId: string,
  ): Promise<SuccessResponseDto> {
    try {
      // Here you retrieve the booking by its DocId (e.g., bookingId)
      // If booking is not found, throw an exception (e.g., NotFoundException)
      
      let stripeIntent: Stripe.PaymentIntent;

      // Here, check if a payment intent already exists for the booking
      // If not, create a new payment intent via Stripe

      const intentMetadata: StripeIntentMetadata = {
        bookingCode: "bookingCode", // Example: Booking code should be passed dynamically
        bookingId: bookingId,
        paymentReceiveId: "paymentReceiveId", // Save paymentReceiveId dynamically
      };

      // Create a payment intent with Stripe
      stripeIntent = await this.stripeService.paymentIntents.create({
        amount: 1000, // Example: amount should be the booking total price in cents
        currency: "usd", // Set currency
        metadata: intentMetadata,
      });

      // Here you save the payment intent details (e.g., stripeIntent) to your database

      const paymentIntentResponse = new PaymentIntentResponseDto();
      paymentIntentResponse.stripeKey = this.stripePublishableKey;
      paymentIntentResponse.stripeSecret = stripeIntent.client_secret || "";
      paymentIntentResponse.status = stripeIntent.status;
      paymentIntentResponse.currency = stripeIntent.currency;

      return new SuccessResponseDto(
        "Payment intent retrieved successfully",
        paymentIntentResponse,
      );
    } catch (error) {
      this.logger.error("Error in getPaymentIntent:", error);
      throw new BadRequestException("Failed to get payment intent");
    }
  }

  async handleStripeWebhook(
    request: RawBodyRequest<Request>,
    signature: string,
  ) {
    try {
      if (!this.stripeWebhookSecret) {
        throw new Error("Stripe webhook secret not found in the configuration.");
      }

      const event = this.stripeService.webhooks.constructEvent(
        request.rawBody as Buffer,
        signature,
        this.stripeWebhookSecret,
      );

      // Handle the different event types sent by Stripe
      switch (event.type) {
        case "payment_intent.created":
          // Here you handle the payment intent created event
          // Example: Update booking status to 'Payment Created'
          break;

        case "payment_intent.succeeded":
          // Here you handle the successful payment event
          // Example: Update booking status to 'Payment Completed'
          break;

        case "payment_intent.payment_failed":
          // Here you handle the failed payment event
          // Example: Update booking status to 'Payment Failed'
          break;

        default:
          this.logger.log("Unhandled event type: " + event.type);
      }

      // Here you update the database based on the event
      // For example, update payment status in your records

      return { received: true };
    } catch (error) {
      this.logger.error("Error handling Stripe webhook event:", error);
      throw new BadRequestException("Error in webhook event processing");
    }
  }
}
```

### Explanation:

- **getPaymentIntent**: This function checks if a payment intent already exists for the booking. If not, it creates a new payment intent in Stripe. It also updates the booking status and payment record in the database.
- **handleStripeWebhook**: This method processes incoming webhook events from Stripe. It updates the payment status and booking status based on the event type, such as payment success or failure.

## 4. The PaymentReceiveModule

In a typical NestJS application, we would include the necessary modules for the database connection and business logic. However, the imports section for the PaymentReceiveModule has been omitted here for clarity, as the focus is on the core logic of payment intent and webhook handling.

```typescript
@Module({
  controllers: [PaymentReceiveController],
  providers: [PaymentReceiveService, PaymentReceiveRepository],
})
export class PaymentReceiveModule {}
```

### Key Notes:

- The `PaymentReceiveService` is injected into the controller to handle all business logic related to payment processing.
- We use Stripe’s official Node.js SDK to interact with Stripe’s API, managing payment intents and webhook events.
- The service includes robust error handling, logging, and status updates to ensure smooth operation in a production environment.

## Conclusion

This implementation provides a clean, professional way to handle Stripe payments in a NestJS application. By managing payment intents and handling webhooks efficiently, we ensure that users can process payments securely, and booking statuses remain synchronized with payment updates.