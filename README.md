Flutter plugin for interacting with iOS StoreKit and Android Billing Library.

Work in progress.

### How this plugin is different from others

The main difference is that instead of providing unified interface for in-app purchases
on iOS and Android, this plugin exposes two separate APIs.

There are several benefits to this approach:

* We can expose complete API interfaces for both platforms, without having to look for highest
  common denominator of those APIs.
* Dart interfaces designed to match native ones most of the time. `StoreKit` for iOS follows
  native interface in 99% of cases. `BillingClient` for Android is very similar as well, but also
  simplifies some parts of native protocol (mostly replaces listeners with Dart `Future`s).
* Developers familiar with native APIs would find it easier to learn. You can simply refer to
  official documentation in most cases to find details about certain method of field.

All Dart code is thoroughly documented with information taken directly from 
Apple Developers website (for StoreKit) and Android Developers website (for BillingClient).

### StoreKit (iOS)

> Plugin currently implements all native APIs except for **downloads**.
> If you are looking for this functionality consider submitting a pull request
> or leaving your :+1: [here](https://github.com/memspace/iap/issues/1).

Interacting with StoreKit in Flutter is almost 100% identical to the native ObjectiveC
interface.

#### Getting products

```dart
final productIds = ['my.product1', 'my.product2'];
final SKProductsResponse response = await StoreKit.instance.products(productIds);
print(response.products); // list of valid [SKProduct]s
print(response.invalidProductIdentifiers) // list of invalid IDs
```

#### App Store Receipt

```dart
// Get receipt path on device
final Uri receiptUrl = await StoreKit.instance.appStoreReceiptUrl;
// Request a refresh of receipt
await StoreKit.instance.refreshReceipt();
```

#### Handling payments and transactions

Payments and transactions are handled within `SKPaymentQueue`.

It is important to set an observer on this queue as early as possible after
your app launch. Observer is responsible for processing all events
triggered by the queue. Create an observer by extending following class:

```dart
abstract class SKPaymentTransactionObserver {
  void didUpdateTransactions(SKPaymentQueue queue, List<SKPaymentTransaction> transactions);
  void didRemoveTransactions(SKPaymentQueue queue, List<SKPaymentTransaction> transactions) {}
  void failedToRestoreCompletedTransactions(SKPaymentQueue queue, SKError error) {}
  void didRestoreCompletedTransactions(SKPaymentQueue queue) {}
  void didUpdateDownloads(SKPaymentQueue queue, List<SKDownload> downloads) {}
  void didReceiveStorePayment(SKPaymentQueue queue, SKPayment payment, SKProduct product) {}
}
```

See API documentation for more details on these methods.

Make sure to implement `didUpdateTransactions` and process all transactions
according to your needs. Typical implementation should normally look like this:

```dart
void didUpdateTransactions(
    SKPaymentQueue queue, List<SKPaymentTransaction> transactions) async {
  for (final tx in transactions) {
    switch (tx.transactionState) {
      case SKPaymentTransactionState.purchased:
        // Validate transaction, unlock content, etc...
        // Make sure to call `finishTransaction` when done, otherwise
        // this transaction will be redelivered by the queue on next application
        // launch.
        await queue.finishTransaction(tx);
        break;
      case SKPaymentTransactionState.failed:
        // ...
        await queue.finishTransaction(tx);
        break;
      // ...
    }
  }
}
```

Before attempting to add a payment always check if the user can actually
make payments:

```dart
final bool canPay = await StoreKit.instance.paymentQueue.canMakePayments();
```

When that's verified and you've set an observer on the queue you can add
payments. For instance:

```dart
final SKProductsResponse response = await StoreKit.instance.products(['my.inapp.subscription']);
final SKProduct product = response.products.single;
final SKPayment = SKPayment.withProduct(product);
await StoreKit.instance.paymentQueue.addPayment(payment);
// ...
// Use observer to track progress of this payment...
```

#### Restoring completed transactions

```dart
await StoreKit.instance.paymentQueue.restoreCompletedTransactions();
/// Optionally implement `didRestoreCompletedTransactions` and 
/// `failedToRestoreCompletedTransactions` on observer to track
/// result of this operation.
```

### BillingClient (Android)

Plugin wraps official [Google Play Billing Library](https://developer.android.com/google/play/billing/billing_library_overview).
Main entry point is the `BillingClient` class:

#### Set listener and start connection

In order to use `BillingClient` we need to start connection with billing service. But before
we can initiate connection we need to set purchase update listener which is required to
handle purchases:

```dart
import 'package:iap/iap.dart';

class MyAppBillingService implements PurchaseUpdatedListener {
  MyAppBillingService() {
    BillingClient.instance.setListener(this);
  }

  bool _connected = false;

  Future<void> _ensureConnected() async {
    if (_connected) return;
    try {
      await BillingClient.instance.startConnection(onDisconnected: _handleDisconnect);
      _connected = true;
    } on BillingClientException catch(error) {
      // Handle exception by checking response code in [error.code].
    }
  }

  void _handleDisconnect() {
    // When client disconnects we get notification here.
    _connected = false;
  }
}
```

Once connection has been established we can start using other functionality of `BillingClient`.
