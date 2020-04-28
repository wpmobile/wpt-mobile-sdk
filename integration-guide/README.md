# Integration Guide

> **Note:** The following guide reflects a beta version of the SDK, and is subject to change prior to general availability of the SDK.

* [Set-up](#set-up)
  * [Pre-requisites](#pre-requisites)
  * [Install the SDK](#install-the-sdk)
  * [Configure the SDK](#configure-the-sdk)
* [Test harness](#test-harness)
* [Payments](#payments)
  * [Make a payment](#make-a-payment)
  * [Settle a payment](#settle-a-payment)
  * [Cancel a payment](#cancel-a-payment)
  * [Query the last payment result](#query-the-last-payment-result)
  * [Query the last payment receipt](#query-the-last-payment-receipt)

## Set-up

### Pre-requisites

#### iOS

In order to use the Worldpay Total Mobile SDK for iOS, you must meet the following requirements:

* Your project must use **XCode 11.3** *(Support for XCode 11.4 coming soon)*
* Your project must target **iOS 9.0** or above
* Swift projects must use **Swift 5.1** *(Support for Swift 5.2 coming soon)*

#### Android

In order to use the Worldpay Total Mobile SDK for Android, you must meet the following requirements:

* Your project must have a **minimum Android version of 24** or above
* Your project must have a **target Android version of 29** or above

### Install the SDK

#### iOS

The Worldpay Total Mobile SDK for iOS is provided as a Framework. See [Releases](../releases#ios) for the available versions.

For each target of your app:

1. Add `WorldpayTotalSDK.framework` to *"Frameworks, Libraries, and Embedded Content"*
2. Add the directory you have put the framework file in to *“Framework Search Paths”*, e.g. `$(PROJECT_DIR)/Frameworks/` (recursive)
3. Add the `Dependencies` directory inside the framework file to *“Import Paths”*, e.g. `$(PROJECT_DIR)/Frameworks/WorldpayTotalSDK.framework/Dependencies/` (recursive)

#### Android

The Worldpay Total Mobile SDK for Android is provided as a Maven dependency.

Add the following in the `build.gradle` of your project to specify our Maven repository where the SDK will be downloaded from:

```python
allprojects {
    repositories {
        maven {
            setUrl("http://ec2-34-246-168-118.eu-west-1.compute.amazonaws.com/artifactory/libs-release/")
            credentials {
                username = <username-provided-by-worldpay>
                password = <password-provided-by-worldpay>
            }
        }
    }
}
```

Then define the Worldpay Total Mobile SDK as a dependency for your app by adding the following in the `build.gradle` of your app, setting `<version>` as the version number you require from [Releases](../releases#android):

```python
dependencies {
    compile(name:'com.worldpay:worldpay-total-sdk:<version>')
}
```

### Configure the SDK

The SDK requires 3 configuration parameters before you can use it to make requests:

1. The host URL - this is the URL where your IPC instance has been deployed, this URL should contain:
  * The protocol (`wss://` or `ws://`)
  * The IP address or domain name (e.g. `192.168.1.32`)
  * The port (e.g. `:443`)
  * A context path (e.g. `/stomp`)
2. The connection timeout - the number of seconds the SDK would wait for a process (such as a payment flow) to complete
3. Your paypoint ID - the ID of the paypoint configured in IPC that you want to target from this mobile device

**Swift**
```swift
Configuration.shared.configure(url: "ws://192.168.1.32:1000/stomp",
                               timeout: 300,
                               paypointId: "PAYPOINT_123")
```

**Kotlin**
```kotlin
Configuration.configure("wss://192.168.1.32:443/stomp",
                        300,
                        "PAYPOINT_123")
```

## Test harness

A test harness is available for the SDK which allows payment flows to be executed without an IPC installation or payment terminals, by simulating typical scenarios.

The SDK test harness is an executable Java application that can be run on a Windows/Linux PC or Mac which the mobile device using the SDK can connect to.

Download the test harness from [Releases](../releases#test-harness), and then run it by executing the following from your command line:

```shell
java -jar worldpay-total-sdk-test-harness-<version>.jar
```

You can check the test harness is running by visiting [http://localhost:8080/actuator/health](http://localhost:8080/actuator/health).


You can configure your SDK instance to connect to the test harness (see **Configure the SDK** above) by providing the host URL as `ws://localhost:8080/stomp`.

To customise the port that the server runs on, add `--server.port=XXXX` after the jar name in the java command. The protocol (`ws://`) and context path (`/stomp`) cannot be customised.

## Payments

### Make a payment

The `PaymentManager.startPayment` method supports all types of payments (sale, pre-auth and refund), and all types of payment instrument; card (including present, not present and keyed), cash and card tokens.

To make a payment, first define the type of payment to make. The types of payment request are:

* `CardSale`
* `CardRefund`
* `CardPreAuthSale`
* `TokenisedCardSale`
* `TokenisedCardRefund`
* `CashSale`
* `CashRefund`

**Swift**
```swift
let paymentRequest = CardSale(merchantTransactionReference: "12345678AB", // A transaction reference defined by the merchant
                              value: 1099, // The value of the payment, in minor currency units
                              type: .cardPresent) // How the card will be presented
```

**Kotlin**
```kotlin
val paymentRequest = CardSale("12345678AB", // A transaction reference defined by the merchant
                              1099, // The value of the payment, in minor currency units
                              CardInteraction.CARD_PRESENT) // How the card will be presented
```

During the payment flow, a number of real-time responses will be returned to your app by the SDK, these include:

* Zero or more `PaymentNotification` - an update on the progress of the payment
* Zero or more `PaymentAction` - an action is required to be performed by your app, possible actions are:
  * *Perform a voice authorisation - the authorisation code from a voice authorisation must be provided*
  * *Input a cashback amount - the amount of cashback requested by the customer must be provided*
  * *Accept/Decline AVS results - confirmation that the AVS results should be accepted must be provided*
  * *Verify the customer's signature - confirmation that the customer's signature is valid must be provided*
* Two `PaymentReceipt` - the content of a receipt to be printed, these will be customer and merchant copies
* One `PaymentResult` - the final result of the payment, containing the outcome, gateway transaction reference and IDs, and details of the card used
* Confirmation that the payment flow has completed and no more messages will be received

In the event an action is requested, the payment flow will pause until the action is completed and the requested data is provided to the SDK.

To receive these responses, define a `PaymentHandler`:

**Swift**
```swift
let paymentHandler: PaymentHandler

let notificationReceived = { notification in
    // Display the text to the POS operator
}
let receiptReceived = { receipt in
    // Send the receipt content to a printer
}
let resultReceived = { result in
    // Handle the result of the payment, e.g. display on screen, or store the payment data
}
let actionRequired = { action in
    // Perform the action

    // Respond to the action, confirming it is complete with the relevant data
    paymentHandler.sendActionConfirmation(action: actionConfirmation)
}
let paymentComplete { () in
    // The payment flow has completed, ready to start the next payment
}

let errorReceived { error in
    // Handle unexpected errors
}

paymentHandler = PaymentHandler(notificationReceived: notificationReceived,
                                 receiptReceived: receiptReceived,
                                 resultReceived: resultReceived,
                                 actionRequired: actionRequired,
                                 paymentComplete: paymentComplete,
                                 errorReceived: errorReceived)
```

**Kotlin**
```kotlin
var paymentHandler: PaymentHandler? = null

val notificationReceived = fun (notification: PaymentNotification) {
    // Display the text to the POS operator
}
val receiptReceived = fun (receipt: PaymentReceipt) {
    // Send the receipt content to a printer
}
val resultReceived = fun (result: PaymentResult) {
    // Handle the result of the payment, e.g. display on screen, or store the payment data
}
val actionRequested = fun (action: PaymentActionRequired) {
    // Perform the action

    // Respond to the action
    paymentHandler?.sendActionConfirmation(actionConfirmation)
}
val paymentComplete = fun () {
    // The payment flow has completed, ready to start the next payment
}

paymentHandler = object : PaymentHandler() {
    override fun onEvent(paymentEvent: PaymentEvent) {
        when (paymentEvent) {
            is PaymentNotification -> notificationReceived(paymentEvent)
            is PaymentReceipt -> receiptReceived(paymentEvent)
            is PaymentResult -> resultReceived(paymentEvent)
            is PaymentActionRequired -> actionRequested(paymentEvent)
            is PaymentComplete -> paymentComplete()
        }
    }

    override fun onErrorReceived(error: UnhandledError) {
        // Handle unexpected errors
    }
}
```


And finally start the payment:

**Swift**
```swift
paymentManager.startPayment(request: paymentRequest
                            handler: paymentHandler)
```

**Kotlin**
```kotlin
paymentManager.startPayment(paymentRequest,
                            paymentHandler)
```

### Settle a payment

If a pre-auth payment is made using `CardPreAuthSale`, it must later be settled in order for the funds to be transferred to the merchant.

To settle a pre-auth payment, you must have the gateway transaction reference, card expiry date, and card number returned in the result of the pre-auth payment. Optionally, you can provide a speicfic amount to settle if you wish to settle less than the previously authorised amount.

During the settlement flow, the following real-time responses will be returned to your app by the SDK:

* One `PaymentResult` - the final result of the payment, containing the outcome, gateway transaction reference and IDs, and details of the card used
* Confirmation that the settlement flow has completed and no more messages will be received

As with making a payment, define a `PaymentHandler` to receive these responses, then send the settle payment request:

**Swift**
```swift
let paymentHandler = ...

let settleRequest = CardSettlement(
                      value: 1099,
                      gatewayTransactionReference: "98765-4321-DF",
                      cardNumber: "123456XXXXXX1234")
                      CardDate(month: 12, year: 21))

paymentManager.settlePayment(request: settleRequest
                             handler: paymentHandler)
```

**Kotlin**
```kotlin
val paymentHandler = ...

val settleRequest = CardSettlement(
                      1099,
                      "98765-4321-DF",
                      "123456XXXXXX1234"
                      PaymentCardDate(12, 21))

paymentManager.settlePayment(settleRequest,
                             paymentHandler)
```

### Cancel a payment

To cancel a previously made card payment, you must have the gateway transaction reference, card expiry date, and card number returned in the result of the payment you wish to cancel.

During the cancel flow, the following real-time responses will be returned to your app by the SDK:

* Two `PaymentReceipt` - the content of a receipt to be printed, these will be customer and merchant copies
* One `PaymentResult` - the final result of the payment, containing the outcome, gateway transaction reference and IDs, and details of the card used
* Confirmation that the cancel flow has completed and no more messages will be received

As with making and settling a payment, define a `PaymentHandler` to receive these responses, then send the cancel payment request:

**Swift**
```swift
let paymentHandler = ...

let cancelRequest = PaymentCancelRequest(
                      originalGatewayTransactionReference: "98765-4321-DF",
                      PaymentInstrument(cardExpiryDate: PaymentCardDate(month: 12, year: 21),
                                        cardNumber: "123456XXXXXX1234"))

paymentManager.cancelPayment(request: cancelRequest
                             handler: paymentHandler)
```

**Kotlin**
```kotlin
val paymentHandler = ...

val cancelRequest = PaymentCancelRequest("98765-4321-DF",
                                         PaymentInstrument(PaymentCardDate(12, 21),
                                                           "123456XXXXXX1234"))

paymentManager.cancelPayment(cancelRequest,
                             paymentHandler)
```

### Query the last payment result

To query the *result* of the last payment, you should pass the merchant transaction reference of the payment.

During the query flow, the following real-time responses will be returned to your app by the SDK:

* One `PaymentResult` - the final result of the payment, containing the outcome, gateway transaction reference and IDs, and details of the card used

As with making, settling or cancelling a payment, define a `PaymentHandler` to receive these responses, then send the query payment request:

**Swift**
```swift
let paymentHandler = ...

let queryRequest = PaymentResultQuery(merchantTransactionReference: "12345678AB")

paymentManager.queryPayment(request: queryRequest
                            handler: paymentHandler)
```

**Kotlin**
```kotlin
val paymentHandler = ...

val queryRequest = PaymentResultQuery("12345678AB")

paymentManager.queryPayment(queryRequest,
                            paymentHandler)
```

### Query the last payment receipt

To query the *receipt* of the last payment, you should pass the merchant transaction reference of the payment and the type of receipt required.

During the query flow, the following real-time responses will be returned to your app by the SDK:

* One `PaymentReceipt` - the content of a receipt to be printed, these will be the customer or merchant copy, depending on your request

As with making, settling or cancelling a payment, define a `PaymentHandler` to receive these responses, then send the query payment request:

**Swift**
```swift
let paymentHandler = ...

let queryRequest = PaymentReceiptQuery(
                     merchantTransactionReference: "12345678AB"
                     type: .customer)

paymentManager.queryPayment(request: queryRequest
                            handler: paymentHandler)
```

**Kotlin**
```kotlin
val paymentHandler = ...

val queryRequest = PaymentReceiptQuery(
                     "12345678AB",
                     ReceiptType.CUSTOMER)

paymentManager.queryPayment(queryRequest,
                            paymentHandler)
```

<!-- Undocumented functionality

## Tax

### Request tax-free voucher

## Reports

### Get batch report

### Get offline transactions report

## Monitoring

### Query IPC status

### Query payment device status
-->

###### ©2020 Worldpay, LLC and/or its affiliates. All rights reserved.
