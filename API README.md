## Documentation

The documentation and demo code will be done in a PHP environment but the concepts should be easily transferable to other 
languages as you'll be interacting with the system through API calls.

### Flow

The user will add items to his/her cart and follow the usual process as setup by the merchant and select PayJustNow
as the payment option on the `checkout` action. Once the user checks out they will be redirected to the PayJustNow 
application to complete the payment process.

Upon reaching the PayJustNow application the user will have to either log in or sign up. Once that's done they will
see a screen detailing the payment plan and will be asked for their card details to initiate and accept the plan.

Depending on the outcome, the user would then be redirected back to a success or failed callback url provided by the
merchant.

### Requirements

`guzzlehttp/guzzle`

### Testing

When testing your integration with PayJustNow you'll have to make use of the `sandbox.payjustnow.com` domain and when 
you're ready to switch production you'll set the domain to `api.payjustnow.com`. The sandbox environment is set to be as
accurate as possible to the production environment.

For authentication, the `user:password` values in the API calls can be replaced with `1:secret` test credentials. 

### Payload

The structure that is expected to be received on `POST sandbox.payjustnow.com/api/v1/merchant/checkout`:

```php
$payload = [
    'customer' => [
        'first_name' => $user->first_name,
        'last_name' => $user->last_name,
        'email' => $user->email,
        'mobile_number' => $user->mobile_number,
        'address' => [
            'address_line_1' => 'address_line_1',
            'address_line_2' => 'address_line_2',
            'city' => 'city',
            'province' => 'Western Cape',
            'postal_code' => 'postal_code'
        ]
    ],
    'order' => [
        'amount' => $order->amount,
        'merchant_reference' => $order->merchant_reference,
        'success_callback_url'=> 'https://yourdomain.com/order-success',
        'fail_callback_url'=> 'https://yourdomain.com/order-fail',
        'items' => $order->items->toArray()
    ]
]
```

Validation for the above fields are as follows:

```php
[
    'customer.first_name' => 'required',
    'customer.last_name' => 'required',
    'customer.email' => 'required',
    'customer.mobile_number' => 'required', // Formatted to E164 ie. +27841112233
    'customer.address.address_line_1' => 'required',
    'customer.address.city' => 'required',
    'customer.address.postal_code' => 'required',
    'customer.address.province' => 'required', // String ie. Western Cape
    'order.merchant_reference' => 'required',
    'order.amount' => 'required|integer|min:1',
    'order.success_callback_url' => 'required',
    'order.fail_callback_url' => 'required',
    'order.items.*.merchant_reference' => 'required',
    'order.items.*.quantity' => 'required|integer',
    'order.items.*.description' => 'required',
    'order.items.*.unit_price' => 'required|integer' // Given in cents 
]
```

### Performing the checkout call

```php
try {
    $res = $client->request(
        'POST',
        'sandbox.payjustnow.com/api/v1/merchant/checkout', [
            'headers' => [
                'Authorization' => 'Basic ' . base64_encode('user:password'),
                'Content-Type'  => 'application/json',
                'Accept'        => 'application/json'
            ],
            'json'            => $payload,
            'allow_redirects' => false
        ]);

    $result = json_decode($res->getBody()->getContents())->data;
} catch (\Throwable $e) {
    // Handle error.
}

$order->payjustnowAttributes()->create([
    'token' => $result->token,
    'name' => 'expires_at',
    'value' => $result->expires_at
]);

header('Location: ' . $result->redirect_to);
```

The `payjustnowAttributes` is a model or table in your application to keep track of requested checkouts on our system.
In the context of Laravel, the table schema would look like the following:

```php
Schema::create('order_payjustnow_attributes', function (Blueprint $table) {
    $table->increments('id');
    $table->integer('order_id');
    $table->string('token');
    $table->string('name');
    $table->string('value');
    $table->timestamps();
});
```

The `header` redirects the user to PayJustNow where the checkout process is completed and the user is then redirected
to your `success_callback_url` or `fail_callback_url`. 

The following query parameters are attached to your `success_callback_url`: `?token=xyz&status=SUCCESS`.

The `success_callback_url` should return a json response containing the following information:

```php
[
    'token' => "",
    'return_url' => "",
]
```

This should complete the payment process.

### Getting the payment configuration for a product

The structure that is expected to be received on `POST sandbox.payjustnow.com/api/v1/merchant/configuration`:

```php
$payload = [
    'amount' => $product->unit_price,
    'customer' => [
        'email' => $user->email
    ]
];
```

### Performing the configuration call

```php
try {
    $res = $client->request(
        'POST',
        'sandbox.payjustnow.com/api/v1/merchant/configuration', [
            'headers' => [
                'Authorization' => 'Basic ' . base64_encode('user:password'),
                'Content-Type'  => 'application/json',
                'Accept'        => 'application/json'
            ],
            'json'            => $payload,
            'allow_redirects' => false
        ]);
} catch (\Throwable $e) {
    // Handle error.
}
```

If an empty payload is sent to the endpoint you would receive the following response:

```json
"data": {
    "minimum_amount": "10000",
    "maximum_amount": "100000",
}
```

Sending the `customer` parameter would result in the same response as above but specific to that user.

Sending the `amount` parameter would result in the payment plan being returned:

```json
"data": {
    "payment_plan": {
        "due_at": ""
        "amount": "",
        "excess": ""
    }
}
```

When including all parameters, the response would be as expected:

```json
"data": {
    "minimum_amount": "10000",
    "maximum_amount": "100000",
    "payment_plan": {
        "due_at": ""
        "amount": "",
        "excess": ""
    }
}
```
