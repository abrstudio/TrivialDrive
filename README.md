# Trivial Drive

## WHAT IS THIS SAMPLE?
This game is a simple "driving" game where the player can buy gas
and drive. The car has a tank which stores gas. When the player receives
gas, the tank fills up (1/4 tank at a time). When the player drives, the gas
in the tank diminishes (also 1/4 tank at a time).

You can receive gas by either watching ads (free gas) or purchasing in-app product (buy gas).

This game has been changed to become a sample for AbrStudio Android Ad & IAB SDK.
![TrivialDrive Screenshot](https://image.ibb.co/dQZFad/Trivial_Drive.jpg)

## HOW TO RUN THIS SAMPLE?
This sample can't be run as-is. Here is what you should do:

1. First you have to contact us for publishing your game in Iran. Visit [AbrStudio Website][website] or send an Email to [info@abrstudio.ir](mailto:info@abrstudio.ir) for more information.

[website]: http://abrstudio.ir "AbrStudio Website"

2. Get your application's credentials (including API key, Sign key and CafeBazaar public key) from us.
3. Change the sample's package name to your package name. To do that, you only need to update the package name in AndroidManifest.xml and correct the references (especially the references to the R object).

## WHAT DOES THIS SAMPLE DO?
This project is a sample for AbrStudio Android Ad & IAB SDK.

### INITIALIZATION
In order to use Ad or IAB SDK you have to first initialize the SDK. The following code shows how TrivialDrive initializes the SDK.


```java
public class MainActivity extends Activity {
	// ...
    
	// Your application's API key that you got from abrstudio.
    private static final String API_KEY = "<PUT YOUR API KEY HERE>";

    // Your application's SIGN key that you got from abrstudio.
    private static final String SIGN_KEY = "<PUT YOUR SIGN KEY HERE>";

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // ...

        AbrStudio.initialize(API_KEY, SIGN_KEY, this);

        // ...
    }
    // ...
}
```

### ADVERTISING SDK
Using advertising sdk has 2 main steps:
1. Requesting Ad: 

    ```java
    // User clicked the "Free Gas" button.
    public void onFreeGasButtonClicked(View arg0) {
        Log.d(TAG, "Free gas button clicked.");

        if (mTank >= TANK_MAX) {
            complain("Your tank is full. Drive around a bit!");
            return;
        }

        // launch the free gas ad flow.
        // We will be notified of request ad result using AbrAdRequestCallback.
        // We will be notified of show ad result using AbrShowAdCallback.
        setWaitScreen(true);
        Log.d(TAG, "Launching request ad flow for free gas.");

        // Make sure you have already initialized AbrStudio
        AbrStudioAd.requestAd(MainActivity.this, "ABR_AD_ZONE_ID", new AbrAdRequestCallback() {
            @Override
            public void onAdReady(AbrStudioAdItem ad) {
				// ...
            }

            @Override
            public void onError(int code, String message) {
                setWaitScreen(false);
                Log.e(TAG, "Error loading ad: code = " + code + ", message = " + message);
            }


            @Override
            public void onNoAdAvailable() {
                setWaitScreen(false);
                Log.e(TAG, "onNoAdAvailable");
            }

            @Override
            public void onNoNetwork() {
                setWaitScreen(false);
                Log.e(TAG, "onNoNetwork");
            }
        });
    }
    ```
2. Showing Ad:

    ```java
    @Override
	public void onAdReady(AbrStudioAdItem ad) {
		ad.show(MainActivity.this, new AbrShowAdCallback() {
            @Override
            public void onError(int code, String message) {
                setWaitScreen(false);
            }

            @Override
            public void onAdClosed(boolean completed) {
                Log.e(TAG, "onAdClosed, completed: " + completed);
                setWaitScreen(false);
            }

            @Override
            public void onReward(AbrStudioAdItem abrStudioAdItem, String reward) {
                Log.e(TAG, "onReward: IN ON REWARD");
                mTank = mTank == TANK_MAX ? TANK_MAX : mTank + 1;
                saveData();
                updateUi();
            }
    	});
	}
    ```
    
For more information about AbrStudio Ad SDK and how to use it, please visit [AbrStudio Website][website].
    
### IN-APP-BILLING SDK
These are main steps 

1. Setup IabHelper:
    ```java
    public class MainActivity extends Activity {
        // ...

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            // ...

            String base64EncodedPublicKey = "";
			
            mHelper = new IabHelper(this, base64EncodedPublicKey);

            // Start setup. This is asynchronous and the specified listener
            // will be called once setup completes.
            // Make sure you have already initialized AbrStudio
            mHelper.startSetup(new IabHelper.OnIabSetupFinishedListener() {
                public void onIabSetupFinished(IabResult result) {
                    if (!result.isSuccess()) {
                        // Oh noes, there was a problem.
                        return;
                    }

                    // IAB is fully set up. Now, let's get an inventory of stuff we own.
                    try {
                        mHelper.queryInventoryAsync(mGotInventoryListener);
                    } catch (IabAsyncInProgressException e) {
                        // ...
                    }
                }
            });
        }
        // ...
    }
    ```
2. Purchase product:
	```java
    // User clicked the "Buy Gas" button
    public void onBuyGasButtonClicked(View arg0) {
        if (mTank >= TANK_MAX) {
            // Tank is full
            return;
        }

        // launch the gas purchase UI flow.
        // We will be notified of completion via mPurchaseFinishedListener
        setWaitScreen(true);
        
        String payload = "";

        try {
            mHelper.launchPurchaseFlow(this, SKU_GAS, RC_REQUEST, mPurchaseFinishedListener, payload);
        } catch (IabAsyncInProgressException e) {
            setWaitScreen(false);
        }
    }
    
    // Callback for when a purchase is finished
    IabHelper.OnIabPurchaseFinishedListener mPurchaseFinishedListener = new IabHelper.OnIabPurchaseFinishedListener() {
        public void onIabPurchaseFinished(IabResult result, Purchase purchase) {
            // if we were disposed of in the meantime, quit.
            if (mHelper == null) return;

            if (result.isFailure()) {
                complain("Error purchasing: " + result);
                setWaitScreen(false);
                return;
            }
            if (!verifyDeveloperPayload(purchase)) {
                complain("Error purchasing. Authenticity verification failed.");
                setWaitScreen(false);
                return;
            }

            Log.d(TAG, "Purchase successful.");

            if (purchase.getSku().equals(SKU_GAS)) {
                // bought 1/4 tank of gas. So consume it.
                Log.d(TAG, "Purchase is gas. Starting gas consumption.");
                try {
                    mHelper.consumeAsync(purchase, mConsumeFinishedListener);
                } catch (IabAsyncInProgressException e) {
                    complain("Error consuming gas. Another async operation in progress.");
                    setWaitScreen(false);
                    return;
                }
            }
        }
    };
    ```
2. Consume:
	```java
    try {
		mHelper.consumeAsync(purchase, mConsumeFinishedListener);
	} catch (IabAsyncInProgressException e) {
		complain("Error consuming gas. Another async operation in progress.");
		setWaitScreen(false);
		return;
	}
    
    // Called when consumption is complete
    IabHelper.OnConsumeFinishedListener mConsumeFinishedListener = new IabHelper.OnConsumeFinishedListener() {
        public void onConsumeFinished(Purchase purchase, IabResult result) {
            // if we were disposed of in the meantime, quit.
            if (mHelper == null) return;

            // We know this is the "gas" sku because it's the only one we consume,
            // so we don't check which sku was consumed. If you have more than one
            // sku, you probably should check...
            if (result.isSuccess()) {
                // successfully consumed, so we apply the effects of the item in our
                // game world's logic, which in our case means filling the gas tank a bit
                mTank = mTank == TANK_MAX ? TANK_MAX : mTank + 1;
                saveData();
            }
            else {
                complain("Error while consuming: " + result);
            }
            updateUi();
            setWaitScreen(false);
        }
    };
    ```
    
For more information about AbrStudio IAB SDK and how to use it, please visit [AbrStudio Website][website].