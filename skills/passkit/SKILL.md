---
name: passkit
description: PassKit and Apple Pay patterns for payment processing, Wallet passes, and in-app purchases via payment sheet. Use when integrating Apple Pay or creating Wallet passes.
---

# PassKit and Apple Pay

Payment processing, Wallet passes, and Apple Pay integration for iOS and macOS apps.

## When This Skill Activates

Use this skill when the user:
- Wants to integrate Apple Pay into their app
- Needs to create or manage Wallet passes (boarding passes, tickets, coupons, loyalty cards)
- Asks about PKPaymentRequest, PKPaymentAuthorizationController, or PayWithApplePayButton
- Needs to check Apple Pay availability or configure merchant IDs
- Wants to sign, distribute, or update .pkpass files
- Asks about payment sheet configuration or supported payment networks
- Needs sandbox testing guidance for Apple Pay

## Decision Tree: Apple Pay vs StoreKit vs PassKit Passes

| Goal | Framework | Key Classes |
|------|-----------|-------------|
| Sell physical goods or services | **PassKit (Apple Pay)** | PKPaymentRequest, PKPaymentAuthorizationController |
| Sell digital content or subscriptions | **StoreKit** | Product, StoreKit.Transaction |
| Boarding passes, tickets, coupons, loyalty cards | **PassKit (Wallet Passes)** | PKPass, PKAddPassesViewController |
| Recurring payments for physical goods | **PassKit (Apple Pay)** + server | PKRecurringPaymentRequest |

> StoreKit is required for digital goods by App Store policy. Apple Pay is for physical goods, services, and donations.

## API Availability

| API | iOS | macOS | watchOS | Notes |
|-----|-----|-------|---------|-------|
| PKPaymentAuthorizationViewController | 8.0+ | -- | -- | UIKit-based payment sheet |
| PKPaymentAuthorizationController | 10.0+ | 11.0+ | 3.0+ | Cross-platform payment sheet |
| PayWithApplePayButton (SwiftUI) | 16.0+ | 13.0+ | -- | SwiftUI native button |
| PKAddPassesViewController | 6.0+ | -- | -- | Add passes to Wallet |
| PKPassLibrary | 6.0+ | -- | 2.0+ | Manage passes in Wallet |

## Apple Pay Setup

### 1. Merchant ID and Certificates

Register a Merchant ID in the Apple Developer portal:
- Identifier format: `merchant.com.yourcompany.appname`
- Create a Payment Processing Certificate (used by your payment processor)
- Create a Merchant Identity Certificate (for server-to-server communication if needed)

### 2. Entitlements and Capabilities

Add the Apple Pay capability in Xcode, which adds the entitlement:

```swift
// Entitlements file entry (added automatically by Xcode)
// com.apple.developer.in-app-payments -> [merchant.com.yourcompany.appname]
```

### 3. Payment Processor Integration

Apple Pay does not process payments directly. You need a payment processor (Stripe, Braintree, Adyen, etc.) that receives the encrypted payment token from Apple Pay and charges the customer.

## Checking Apple Pay Availability

Always check before showing Apple Pay UI:

```swift
import PassKit

func canUseApplePay() -> Bool {
    // Check if the device supports Apple Pay at all
    guard PKPaymentAuthorizationController.canMakePayments() else {
        return false
    }

    // Check if the user has cards in the specified networks
    let supportedNetworks: [PKPaymentNetwork] = [.visa, .masterCard, .amex, .discover]
    return PKPaymentAuthorizationController.canMakePayments(
        usingNetworks: supportedNetworks
    )
}
```

If `canMakePayments()` is true but `canMakePayments(usingNetworks:)` is false, show a "Set Up Apple Pay" button using `PKPaymentButton(paymentButtonType: .setUp, paymentButtonStyle: .black)`.

## PKPaymentRequest Configuration

```swift
func createPaymentRequest() -> PKPaymentRequest {
    let request = PKPaymentRequest()
    request.merchantIdentifier = "merchant.com.yourcompany.appname"
    request.supportedNetworks = [.visa, .masterCard, .amex, .discover]
    request.merchantCapabilities = [.capability3DS, .capabilityDebit, .capabilityCredit]
    request.countryCode = "US"
    request.currencyCode = "USD"

    // Payment summary items (last item is the total)
    request.paymentSummaryItems = [
        PKPaymentSummaryItem(label: "Widget", amount: NSDecimalNumber(string: "9.99")),
        PKPaymentSummaryItem(label: "Shipping", amount: NSDecimalNumber(string: "2.99")),
        PKPaymentSummaryItem(label: "Your Company", amount: NSDecimalNumber(string: "12.98"))
    ]

    // Optional: require shipping or billing info
    request.requiredShippingContactFields = [.postalAddress, .emailAddress, .phoneNumber]
    request.requiredBillingContactFields = [.postalAddress]

    return request
}
```

### Payment Summary Items

The last item in `paymentSummaryItems` is always the **total** displayed on the payment sheet. Its `label` is your company or app name.

```swift
// Pending amounts (e.g., estimated shipping)
let shipping = PKPaymentSummaryItem(
    label: "Estimated Shipping",
    amount: NSDecimalNumber(string: "5.00"),
    type: .pending  // Shows as "pending" on the sheet
)
```

## UIKit: PKPaymentAuthorizationViewController

```swift
import PassKit
import UIKit

class CheckoutViewController: UIViewController, PKPaymentAuthorizationViewControllerDelegate {

    func startApplePay() {
        let request = createPaymentRequest()

        guard let controller = PKPaymentAuthorizationViewController(paymentRequest: request) else {
            // Payment request is invalid or Apple Pay unavailable
            return
        }
        controller.delegate = self
        present(controller, animated: true)
    }

    // Called when the user authorizes with Face ID / Touch ID / passcode
    func paymentAuthorizationViewController(
        _ controller: PKPaymentAuthorizationViewController,
        didAuthorizePayment payment: PKPayment,
        handler completion: @escaping (PKPaymentAuthorizationResult) -> Void
    ) {
        // Send payment.token.paymentData to your server / payment processor
        processPaymentWithServer(token: payment.token) { success in
            if success {
                completion(PKPaymentAuthorizationResult(status: .success, errors: nil))
            } else {
                let error = PKPaymentRequest.paymentShippingAddressUnserviceableError(
                    withLocalizedDescription: "Payment failed. Please try again."
                )
                completion(PKPaymentAuthorizationResult(status: .failure, errors: [error]))
            }
        }
    }

    func paymentAuthorizationViewControllerDidFinish(
        _ controller: PKPaymentAuthorizationViewController
    ) {
        controller.dismiss(animated: true)
    }

    private func processPaymentWithServer(
        token: PKPaymentToken,
        completion: @escaping (Bool) -> Void
    ) {
        // Send token.paymentData to your payment processor
        completion(true)
    }
}
```

## Cross-Platform: PKPaymentAuthorizationController

Use `PKPaymentAuthorizationController` for macOS, watchOS, or when you want a non-view-controller-based API:

```swift
import PassKit

func presentApplePayController() {
    let request = createPaymentRequest()
    let controller = PKPaymentAuthorizationController(paymentRequest: request)
    controller.delegate = self
    controller.present()
}

// PKPaymentAuthorizationControllerDelegate
func paymentAuthorizationController(
    _ controller: PKPaymentAuthorizationController,
    didAuthorizePayment payment: PKPayment,
    handler completion: @escaping (PKPaymentAuthorizationResult) -> Void
) {
    // Same token handling as the ViewController version
    completion(PKPaymentAuthorizationResult(status: .success, errors: nil))
}

func paymentAuthorizationControllerDidFinish(
    _ controller: PKPaymentAuthorizationController
) {
    controller.dismiss()
}
```

## SwiftUI: PayWithApplePayButton

Available in iOS 16+ / macOS 13+:

```swift
import SwiftUI
import PassKit

struct CheckoutView: View {
    @State private var paymentStatus: String = ""

    var body: some View {
        VStack(spacing: 20) {
            Text("Total: $12.98")
                .font(.title2)

            PayWithApplePayButton(.checkout) {
                let request = createPaymentRequest()
                let controller = PKPaymentAuthorizationController(paymentRequest: request)
                controller.delegate = makeDelegate()
                controller.present()
            }
            .frame(height: 45)
            .payWithApplePayButtonStyle(.black)

            if !paymentStatus.isEmpty {
                Text(paymentStatus)
            }
        }
        .padding()
    }
}
```

## Wallet Passes

### PKPass and PKAddPassesViewController

```swift
import PassKit
import UIKit

class PassViewController: UIViewController {

    func addPassFromURL(_ url: URL) {
        guard let passData = try? Data(contentsOf: url),
              let pass = try? PKPass(data: passData) else {
            return
        }

        guard let addController = PKAddPassesViewController(pass: pass) else {
            return
        }
        present(addController, animated: true)
    }

    func addMultiplePasses(_ passDataArray: [Data]) {
        let passes = passDataArray.compactMap { try? PKPass(data: $0) }
        guard !passes.isEmpty,
              let addController = PKAddPassesViewController(passes: passes) else {
            return
        }
        present(addController, animated: true)
    }
}
```

### pass.json Structure

A `.pkpass` bundle is a signed ZIP archive containing:

```
MyPass.pkpass/
  pass.json          # Required: pass definition
  icon.png           # Required: pass icon
  icon@2x.png
  logo.png           # Optional: logo on the pass
  strip.png          # Optional: strip image (event tickets, coupons)
  manifest.json      # Generated: SHA-1 hashes of all files
  signature           # Generated: PKCS#7 detached signature
```

Example `pass.json` for a boarding pass:

```json
{
  "formatVersion": 1,
  "passTypeIdentifier": "pass.com.yourcompany.boardingpass",
  "serialNumber": "ABC123",
  "teamIdentifier": "YOUR_TEAM_ID",
  "organizationName": "Your Airline",
  "description": "Boarding Pass",
  "foregroundColor": "rgb(255, 255, 255)",
  "backgroundColor": "rgb(0, 100, 200)",
  "boardingPass": {
    "headerFields": [
      { "key": "gate", "label": "Gate", "value": "B12" }
    ],
    "primaryFields": [
      { "key": "origin", "label": "SFO", "value": "San Francisco" },
      { "key": "destination", "label": "JFK", "value": "New York" }
    ],
    "secondaryFields": [
      { "key": "boarding", "label": "Boarding", "value": "10:30 AM" }
    ],
    "auxiliaryFields": [
      { "key": "seat", "label": "Seat", "value": "14A" }
    ],
    "backFields": [
      { "key": "info", "label": "Details", "value": "Flight AA100" }
    ]
  },
  "barcode": {
    "format": "PKBarcodeFormatQR",
    "message": "ABC123-BOARDING",
    "messageEncoding": "iso-8859-1"
  }
}
```

Pass types: `boardingPass`, `coupon`, `eventTicket`, `generic`, `storeCard`.

### Signing Passes

Passes must be signed server-side using a Pass Type ID certificate from the Apple Developer portal. Use Apple's `signpass` tool or a server-side library (Ruby, Node.js, Python).

### Updating Passes with Push Notifications

To update passes after they are added to Wallet:

1. Include `webServiceURL` and `authenticationToken` in pass.json
2. Implement a REST API at that URL for pass registration and updates
3. When your data changes, send a push notification to the registered device
4. The device calls your API to fetch the updated pass

```json
{
  "webServiceURL": "https://api.yourcompany.com/passes/",
  "authenticationToken": "a_secure_token_here"
}
```

Required server endpoints:
- `POST /devices/{deviceId}/registrations/{passTypeId}/{serialNumber}` -- register
- `DELETE /devices/{deviceId}/registrations/{passTypeId}/{serialNumber}` -- unregister
- `GET /devices/{deviceId}/registrations/{passTypeId}` -- list updated passes
- `GET /passes/{passTypeId}/{serialNumber}` -- fetch updated pass

## Testing Apple Pay in Sandbox

1. Create a sandbox tester account in App Store Connect
2. Sign into the sandbox account on a test device (Settings > App Store > Sandbox Account)
3. Add sandbox test cards in Wallet (Apple provides test card numbers)
4. Use the sandbox environment in your payment processor
5. Real cards are never charged in sandbox mode

Test card numbers vary by payment processor. Check your processor's documentation for sandbox-specific card numbers.

**Simulator limitations**: The iOS Simulator supports Apple Pay UI for layout testing but cannot complete real payment authorization. Always test on a physical device for end-to-end payment flows.

## Common Pitfalls and Patterns

### Payment Request

```swift
// ✅ Good: Last summary item is the total with your company name
request.paymentSummaryItems = [
    PKPaymentSummaryItem(label: "Blue Widget", amount: NSDecimalNumber(string: "9.99")),
    PKPaymentSummaryItem(label: "Shipping", amount: NSDecimalNumber(string: "2.99")),
    PKPaymentSummaryItem(label: "Your Company", amount: NSDecimalNumber(string: "12.98"))
]

// ❌ Bad: Missing total item or mismatched amounts
request.paymentSummaryItems = [
    PKPaymentSummaryItem(label: "Blue Widget", amount: NSDecimalNumber(string: "9.99"))
]
```

### Availability Check

```swift
// ✅ Good: Check both device support and card availability
if PKPaymentAuthorizationController.canMakePayments(usingNetworks: supportedNetworks) {
    showApplePayButton()
} else if PKPaymentAuthorizationController.canMakePayments() {
    showSetUpApplePayButton()
} else {
    showTraditionalCheckout()
}

// ❌ Bad: Showing Apple Pay without checking availability
showApplePayButton() // May crash or show empty sheet
```

### Payment Token Handling

```swift
// ✅ Good: Send the encrypted token to your server, never decode client-side
func didAuthorizePayment(_ payment: PKPayment, handler: @escaping (PKPaymentAuthorizationResult) -> Void) {
    let tokenData = payment.token.paymentData
    sendToPaymentProcessor(tokenData) { result in
        handler(PKPaymentAuthorizationResult(status: result ? .success : .failure, errors: nil))
    }
}

// ❌ Bad: Trying to decode the payment token on the client
let json = try JSONSerialization.jsonObject(with: payment.token.paymentData) // Never do this
```

### Decimal Precision

```swift
// ✅ Good: Use string-based NSDecimalNumber for currency
let amount = NSDecimalNumber(string: "19.99")

// ❌ Bad: Using floating-point (precision loss with currency)
let amount = NSDecimalNumber(value: 19.99) // May become 19.9899999...
```

### Dismiss Handling

```swift
// ✅ Good: Always dismiss in didFinish, regardless of success or cancel
func paymentAuthorizationViewControllerDidFinish(_ controller: PKPaymentAuthorizationViewController) {
    controller.dismiss(animated: true)
}

// ❌ Bad: Dismissing only on success (leaks the payment sheet on cancel)
```
