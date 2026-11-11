# Paddle Checkout

Accept one-time payments on WemX through Paddle Billing (Merchant of Record) — card, PayPal, and local payment methods, with Paddle handling global tax/VAT compliance.

## Features

- Create a Paddle transaction and open Paddle's checkout overlay for the customer
- Self-configures the Paddle account's default checkout URL via the API on first use — no manual Paddle Dashboard step beyond entering API keys
- Confirm payment from a signed `transaction.completed` webhook
- Full and partial refunds via Paddle's Adjustments API

## Install

Install from the WemX marketplace, or download `PaddleCheckout.zip` from a GitHub release and place the `PaddleCheckout` folder at `extensions/Gateways/PaddleCheckout`. Then enable **Paddle Checkout**.

Publishing a release builds `PaddleCheckout.zip`. Unzipping it creates a folder named `PaddleCheckout`. GitHub's own "Source code" archive still unpacks to `gateway-paddle-checkout-<tag>`. Use `PaddleCheckout.zip`.

Unlike most gateways, this one also needs two small additions to WemX core, because **Paddle Billing's checkout is a client-side overlay (Paddle.js), not a plain hosted page you can redirect to** — Paddle has no "checkout page URL" to send customers to directly.

1. Copy `resources/paddle-checkout.blade.php` from this release zip to `resources/client_area/default/payments/paddle-checkout.blade.php` (use your active theme's `payments/` folder if not on `default`).

2. Add this route inside the `Route::middleware(['web'])->group(...)` block in `routes/web.php` (it already contains `payments.gateway.webhook` and `payments.gateway.callback`):

   ```php
   Route::get('/payments/paddle/checkout', [Client\PaymentsController::class, 'paddleCheckout'])->name('payments.paddle.checkout');
   ```

3. Add this method to `app/Http/Controllers/Client/PaymentsController.php`:

   ```php
   /**
    * Paddle's "default payment link" destination. Paddle appends ?_ptxn={transaction_id}
    * to this URL once a transaction is created, and Paddle.js (embedded in the view)
    * reads that parameter to open the actual checkout overlay for the customer.
    */
   public function paddleCheckout(Request $request)
   {
       $transactionId = (string) $request->query('_ptxn', '');

       abort_if($transactionId === '', 404);

       $payment = Payment::where('transaction_id', $transactionId)->first();

       abort_if(! $payment, 404);

       $gatewayConfig = $payment->gatewayConfig;

       abort_if(! $gatewayConfig, 404);

       return view('theme::payments.paddle-checkout', [
           'transactionId' => $transactionId,
           'sandbox' => (string) $gatewayConfig->config('sandbox', '0') === '1',
           'clientSideToken' => $gatewayConfig->config('client_side_token'),
           'callbackUrl' => route('payments.gateway.callback', [
               'webhook_id' => $gatewayConfig->webhook_id,
               'payment_token' => $payment->token,
           ]),
       ]);
   }
   ```

   `Payment`, `GatewayConfig`, and `Request` should already be imported at the top of this controller.

   If a companion recurring-billing gateway is installed alongside this one and has already added this route/method, skip these two steps — they share the same checkout page.

## Connection

The gateway configuration needs a **Paddle API Key** and a **Client-side Token** (Paddle Dashboard → Developer Tools → Authentication — these are two separate, differently-scoped credentials), a **Sandbox Mode** toggle, and a **Notification Secret Key**. Point a Paddle webhook destination at the gateway webhook URL WemX shows for this configuration, subscribed to the `transaction.completed` event.
