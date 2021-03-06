# Integrating Android Pay with Judopay Android SDK

##### NOTE: Android Pay has launched in the UK. This sample app provides you with a look at how you can integrate Android Pay into your app using Judo's Android SDK.

Android Pay™ is a mobile payment solution that offers further simplicity, security and choice when making purchases with Android phones.

With Android Pay merchants can accept payments for physical goods and services, and access buyers' shipping and payment information stored in their Google account. The judoNative Android SDK can be used to process Android Pay payments made in your app making the checkout journey even simpler and more secure.

In this guide we'll walk you through how to integrate Android Pay with the judoNative Android SDK.

## Requirements
Android Pay is compatible with devices running Android OS 4.4 (KitKat) or later. In addition to this, please check that you have a public key, received when setting up your judo account to use with Android Pay.

## Application Flow
When taking a payment using Android Pay the following steps are involved:
  1. The user initiates a payment, and a check is made to see if the user has Android Pay.
  2. A Masked Wallet Request is created to show the payment button.
  3. The Masked Wallet Response is received when the user has confirmed to pay with Android Pay.
  4. A Full Wallet Request is made to request the Android Pay encrypted payload.
  5. With the Full Wallet Response, a request is made using the judoNative Android SDK to complete the payment.

## Setting up Google Play Services
The Google Play Services Wallet library allows your app to call Android Pay and request the user to make a payment. To use this library, add a new dependency in the build.gradle file of your app module:
```groovy
dependencies {
	compile 'com.google.android.gms:play-services-wallet:10.2.0'
}
```

Next, add the metadata attributes to your app's AndroidManifest.xml to enable the Wallet API:
```xml
<meta-data
   android:name="com.google.android.gms.version"
   android:value="@integer/google_play_services_version" />

<meta-data
   android:name="com.google.android.gms.wallet.api.enabled"
   android:value="true" />
```

## Authenticating with Android Pay
To authenticate your app with Android Pay, you must provide the SHA1 fingerprint of your app's certificate in the Google Developers Console:
  1. Check if your app is signed with a certificate keystore for releasing to Google Play.
  2. Run the gradle task ```signingReport``` to display the SHA1 fingerprint for your app.
  3. Create a new project in the [Google Developers Console](https://console.developers.google.com/) with an OAuth Client ID using the SHA1 fingerprint.
See the [Android Pay tutorial](https://developers.google.com/android-pay/android/tutorial) for more information.

## Requesting Android Pay
#### 1. Check if Android Pay is available on the device
To check if Android Pay is available we'll first need to create a GoogleApiClient, this will be used for all calls to Google Play APIs:
```java
GoogleApiClient googleApiClient = new GoogleApiClient.Builder(this)
    .addApi(Wallet.API, new Wallet.WalletOptions.Builder()
        .setEnvironment(WalletConstants.ENVIRONMENT_PRODUCTION)
        .build())
    .enableAutoManage(this, this)
    .build();
```

Use the ```isReadyToPay()``` method to check if the user has Android Pay and a card is available to make a payment:
```java
Wallet.Payments.isReadyToPay(googleApiClient).setResultCallback(new ResultCallback<BooleanResult>() {
   @Override
   public void onResult(BooleanResult result) {
       if (result.getStatus().isSuccess() && result.getValue()) {
	       // show the pay button
       }
   }
});
```
#### 2. Show the Android Pay button
The Android Pay button is shown by creating a SupportWalletFragment, configured with your judo Android Pay public key and a Masked Wallet Request with the details of the payment:
```java
PaymentMethodTokenizationParameters parameters = PaymentMethodTokenizationParameters.newBuilder()
		.setPaymentMethodTokenizationType(PaymentMethodTokenizationType.NETWORK_TOKEN)
	    .addParameter("publicKey", getString(R.string.public_key))
	    .build();
```
Create a WalletFragmentOptions instance to configure the appearance of the Android Pay button:
```java
WalletFragmentOptions options = WalletFragmentOptions.newBuilder()
       .setEnvironment(WalletConstants.ENVIRONMENT_PRODUCTION)
       .setTheme(WalletConstants.THEME_LIGHT)
       .setMode(WalletFragmentMode.BUY_BUTTON)
       .build();
```
Create a ```SupportWalletFragment``` instance and initialize it with a Masked Wallet Request:
```java
SupportWalletFragment walletFragment = SupportWalletFragment.newInstance(options);
MaskedWalletRequest walletRequest = MaskedWalletRequest.newBuilder()
       .setMerchantName("Sample App")
       .setCurrencyCode("GBP")
       .setEstimatedTotalPrice("5.00")
       .setPaymentMethodTokenizationParameters(parameters)
       .setCart(Cart.newBuilder()
               .setCurrencyCode("GBP")
               .setTotalPrice("5.00")
               .build())
       .build();
WalletFragmentInitParams startParams = WalletFragmentInitParams.newBuilder()
       .setMaskedWalletRequest(walletRequest)
       .setMaskedWalletRequestCode(MASKED_WALLET_REQUEST)
       .build();

walletFragment.initialize(startParams);
```
Add the SupportWalletFragment to your layout:
```java
getSupportFragmentManager()
	.beginTransaction()
    .replace(R.id.my_layout, walletFragment)
    .commit();
```
#### 3. Receiving the MaskedWallet
Once the user has confirmed to make a payment with Android Pay, you will receive a MaskedWallet with the masked payment information and address details. Override the ```onActivityResult``` method in your Activity to receive the MaskedWallet:
```java
@Override
public void onActivityResult(int requestCode, int resultCode, Intent data) {
	super.onActivityResult(requestCode, resultCode, data);

    switch (requestCode) {
	case MASKED_WALLET_REQUEST:
		if (resultCode == Activity.RESULT_OK && data != null) {
			MaskedWallet maskedWallet = data.getParcelableExtra(WalletConstants.EXTRA_MASKED_WALLET);
			// request FullWallet
		}
    }
}
```
#### 4. Requesting the FullWallet
To request the FullWallet containing the encrypted payment token, perform a FullWalletRequest with the payment amount and Google Transaction ID:
```java
FullWalletRequest fullWalletRequest = FullWalletRequest.newBuilder()
	.setGoogleTransactionId(maskedWallet.getGoogleTransactionId())
	.setCart(Cart.newBuilder()
		.setCurrencyCode("GBP")
		.setTotalPrice("5.00")
		.build())
	.build();

Wallet.Payments.loadFullWallet(googleApiClient, fullWalletRequest, FULL_WALLET_REQUEST);
```
#### 5. Receiving the FullWallet
The response from the FullWalletRequest will be received in the Activity result:
```java
@Override
public void onActivityResult(int requestCode, int resultCode, Intent data) {		
  super.onActivityResult(requestCode, resultCode, data);

  switch (requestCode) {
    case FULL_WALLET_REQUEST:
      if (resultCode == Activity.RESULT_OK && data.hasExtra(WalletConstants.EXTRA_FULL_WALLET)) {
        FullWallet fullWallet = data.getParcelableExtra(WalletConstants.EXTRA_FULL_WALLET);
        // call judoNative SDK to perform payment
      }
      break;
  }
}
```

## Using the Judopay SDK with Android Pay

The final step is to make a request to judo with the response received from Android Pay. To do this, create an AndroidPayRequest using the Builder with the required fields. For the public key, use the key received when setting up your judo account to use Android Pay.
```java
AndroidPayRequest androidPayRequest = new AndroidPayRequest.Builder()
        .setJudoId("1234567")
        .setCurrency(Currency.GBP)
        .setAmount(new BigDecimal("1.00"))
        .setWallet(new com.judopay.model.Wallet.Builder()
                .setPublicKey("<YOUR_ANDROID_PAY_PUBLIC_KEY>")
                .setEnvironment(WalletConstants.ENVIRONMENT_PRODUCTION)
                .setPaymentMethodToken(wallet.getPaymentMethodToken().getToken())
                .setGoogleTransactionId(wallet.getGoogleTransactionId())
                .setInstrumentDetails(wallet.getInstrumentInfos()[0].getInstrumentDetails())
                .setInstrumentType(wallet.getInstrumentInfos()[0].getInstrumentType())
                .build())
        .build();
```

To perform the transaction, pass the ```AndroidPayRequest``` to the judoNative API service. This will perform the transaction and charge the customers payment method.

#### Performing a Payment
```java
JudoApiService apiService = Judo.getApiService(this);
apiService.androidPayPayment(androidPayRequest)
        .subscribe(new Action1<Receipt>() {
            @Override
            public void call(Receipt receipt) {
                // handle successful payment
            }
        }, new Action1<Throwable>() {
            @Override
            public void call(Throwable throwable) {
                // show error message
            }
        });
```

#### Performing a Pre-auth
```java
JudoApiService apiService = Judo.getApiService(this);
apiService.androidPayPreAuth(androidPayRequest)
        .subscribe(new Action1<Receipt>() {
            @Override
            public void call(Receipt receipt) {
                // handle successful pre auth
            }
        }, new Action1<Throwable>() {
            @Override
            public void call(Throwable throwable) {
                // show error message
            }
        });
```

## Going Live
When you're ready to go live, you will need to update your code to use the production environment for Android Pay and the judoNative SDK:
 - When calling the Google Play Wallet library, ensure that ```WalletConstants.ENVIRONMENT_PRODUCTION``` is used instead of ```WalletConstants.ENVIRONMENT_TEST```. 
 - When calling the judoNative SDK, ensure that the LIVE environment is set using: ```Judo.setEnvironment(Judo.LIVE);```

All code shown is available to view on our [Android Pay sample app GitHub repo](https://github.com/JudoPay/Judo-AndroidPay-Sample).

If you have any questions, please get in touch at [developersupport@judopayments.com](mailto:developersupport@judopayments.com)
