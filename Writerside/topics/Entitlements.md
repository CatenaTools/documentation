# Entitlements

## How does Catena define an entitlement?

An entitlement is simply an `entitlement_id` - a GUID generated upon the creation of the entitlement, an `entitlement_name` - a unique name entered by an admin while creating the entitlement, and an `entitlement_description` - a description of the entitlement also entered by an admin when creating the entitlement.

Each entitlement also has one or more `entitlement_granting_purchases` associated with it.

Each `entitlement_granting_purchase` specifies a product that a user can purchase from one of our supported In-App Purchase (IAP) stores that will grant them the entitlement.

As such, an `entitlement_granting_purchase` is composed of an `entitlement_granting_purchase_id`, an `iap_store`, an `external_product_id`, and an `entitlement_id`.

If a user purchases the product uniquely identified in `iap_store` by `external_product_id`, they are then granted the entitlement uniquely identified by `entitlement_id`.

**Note:** `entitlement_granting_purchase_id` **is simply a unique identifier for this mapping in our system.**

## How are entitlements created, updated, and deleted?

The short answer - via an admin dashboard. Let’s take a deeper look at how this works.

![Untitled](Untitled.png)

This will be the default view an admin will see when they navigate to the entitlements admin dashboard. All existing entitlements that have previously been created are listed with buttons allowing admins to edit or delete them. The list is populated via `AdminGetAllEntitlements`.

```protobuf
rpc AdminGetAllEntitlements (AdminGetAllEntitlementsRequest) returns (AdminGetEntitlementsResponse) {
    option (google.api.http) = {
		    get: "/api/v1/entitlements"
		};
}
message AdminGetAllEntitlementsRequest {}
// shared between both GET RPCs
message AdminGetEntitlementsResponse {
    repeated Entitlement entitlements = 1;
}
```

### Search Functionality

**(#2)** An admin may also search for a specific entitlement by name e.g. `ANNUAL_SUBSCRIPTION` or for a subset of existing entitlements by keyword e.g. `SUBSCRIPTION` continuing with the example shown above. The search term used should be typed into the search bar, and when the `Search` button is clicked, the list displayed will be repopulated via `AdminGetEntitlementsByApproxName`.

```protobuf
// should only be called when an admin clicks a search button so we are not making a request with each character typed into a search bar
rpc AdminGetEntitlementsByApproxName (AdminGetEntitlementsByApproxNameRequest) returns (AdminGetEntitlementsResponse) {
    option (google.api.http) = {
	      get: "/api/v1/entitlements/{approx_entitlement_name}"
		};
}
message AdminGetEntitlementsByApproxNameRequest {
    string approx_entitlement_name = 1;
}
// shared between both GET RPCs
message AdminGetEntitlementsResponse {
    repeated Entitlement entitlements = 1;
}
```

### Entitlement Creation

********(#1)******** To create a new entitlement, an admin may click the `Create Entitlement` button to be presented with the following form:

![create-entitlement-view.PNG](create-entitlement-view.png)

Here, they can enter an Entitlement Name, an Entitlement Description, and one or more Entitlement Granting Purchases. When the `Save` button is clicked, `AdminCreateEntitlement` will be used to store the new entitlement in `entitlements.entitlements` and `entitlements.entitlement_granting_purchases` *([database].[table])* after generating and assigning it a unique identifier.

**********************Note: The External Product IDs that this form expects can be found in an IAP store’s admin dashboard after following their process to create a new product.**********************

```protobuf
rpc AdminCreateEntitlement (AdminCreateEntitlementRequest) returns (AdminCreateEntitlementResponse) {
    option (google.api.http) = {
		    post: "/api/v1/entitlements"
			  body: "*"
		};
}
message AdminCreateEntitlementRequest {
	string entitlement_name = 1;
	string entitlement_description = 2;
	repeated EntitlementGrantingPurchase entitlement_granting_purchases = 3;
}
message AdminCreateEntitlementResponse {}
```

### Updating Entitlements

********(#3)******** To update an existing entitlement, an admin may click the `Edit` button next to said entitlement in the list displayed in the dashboard. This will present them will the following form:

![update-entitlement-view.PNG](update-entitlement-view.png)

Here, they have the ability to update the Entitlement Name, Entitlement Description, and Entitlement Granting Purchases for this entitlement.

***********************************Note: The unique identifier for an entitlement cannot be changed by an admin.***********************************

************************************************************Note: If an admin removes an entitlement granting purchase - say, external product ID **prod_123** from IAP store **Stripe**, a user who has previously been granted this entitlement by purchasing product **prod_123** from ************Stripe************ will ******not****** have their entitlement revoked.*

When an admin clicks the `Save` button, the information stored in `entitlements.entitlements` and `entitlements.entitlement_granting_purchases` will be updated for this entitlement via `AdminUpdateEntitlement`.

```protobuf
rpc AdminUpdateEntitlement (AdminUpdateEntitlementRequest) returns (AdminUpdateEntitlementResponse) {
    option (google.api.http) = {
		    put: "/api/v1/entitlements"
			  body: "*"
		};
}
message AdminUpdateEntitlementRequest {
	Entitlement entitlement = 1; // need an entitlement_id unlike AdminCreateEntitlementRequest
}
message AdminUpdateEntitlementResponse {}
message Entitlement {
	string entitlement_id = 1; // GUID
	string entitlement_name = 2;
	string entitlement_description = 3;
	repeated EntitlementGrantingPurchase entitlement_granting_purchases = 4; // references to IAP store purchases that grant this entitlement
}
message EntitlementGrantingPurchase {
	string entitlement_granting_purchase_id = 1;
	string external_product_id = 2;
	IAPStore iap_store = 3;
}
// supported In-App Purchase (IAP) stores
enum IAPStore {
	IAP_STORE_UNSPECIFIED = 0;
	IAP_STORE_STRIPE = 1;
}
```

### Entitlement Deletion

**(#4)** To delete an entitlement, an admin may click the `Delete` button next to said entitlement in the  list displayed in the dashboard.

Once they click `Delete`, the entitlement will be deleted from the system, and all users who were previously granted the entitlement will have it revoked from their accounts.

After clicking `Delete`, an admin will have to confirm the deletion of the entitlement by entering its name and clicking the `Confirm Deletion` button. This step helps prevent accidental deletion.

![delete-entitlement-confirmation.PNG](delete-entitlement-confirmation.png)

When an admin clicks the `Confirm Deletion` button, the `Delete` operation is facilitated by the use of `AdminDeleteAndRevokeEntitlement`.

```protobuf
/* Deletes the specified entitlement from our system and revokes it from all users to whom it has previously been granted. */
rpc AdminDeleteAndRevokeEntitlement (AdminDeleteEntitlementRequest) returns (AdminDeleteEntitlementResponse) {
    option (google.api.http) = {
		    delete: "/api/v1/entitlements/revoke/{entitlement_id}"
		};
}

message AdminDeleteAndRevokeEntitlementRequest {
	string entitlement_id = 1;
}
message AdminDeleteAndRevokeEntitlementResponse {}
```

## How can a user purchase a product that will grant them an entitlement?

We will have a checkout page for each of our supported IAP stores.

A user can navigate to one of these checkout pages, authenticate with a Catena account, select the product they wish to purchase, enter their payment information, and view the status of their payment from a page they are redirected to after their payment has been submitted.

When a user purchases a product, the checkout page will make a request to `CreateIAPStorePurchase` whose method handler processes the purchase.

```protobuf
/* Allows an account to purchase a product from a supported IAP store.
 * This purchase can either be a one-time purchase that the account is billed for once
 * or a subscription purchase that the account is billed for on a recurring basis.
 */
rpc CreateIAPStorePurchase (CreateIAPStorePurchaseRequest) returns (CreateIAPStorePurchaseResponse) {
    option (google.api.http) = {
		    post: "/api/v1/entitlements/purchase"
			  body: "*"
		};
}
message CreateIAPStorePurchaseRequest {
    IAPStore iap_store = 1;
	  oneof purchase_request_data {
		    StripePurchaseRequestData stripe_purchase_request_data = 2;
	  }
}
message CreateIAPStorePurchaseResponse {
    IAPStore iap_store = 1;
	  oneof purchase_response_data {
		    StripePurchaseResponseData stripe_purchase_response_data = 2;
	  }
}
message StripePurchaseRequestData {
    string price_id = 1; // Stripe-generated unique ID of the pricing plan for the product being purchased
}
message StripePurchaseResponseData {
    string client_secret = 1; // used for browser checkout session
}
```

A different payload will need to be sent in a `CreateIAPStorePurchaseRequest` for each IAP store, and similarly, `CreateIAPStorePurchaseResponses` returned will contain different payloads. This is due to differences in third-party APIs specific to each IAP store.

For example, when a user purchases a product from Stripe, we will create a new Stripe Subscription for them. The third-party API function to create a new Stripe Subscription requires a `price_id` and a `customer_id`. This differs from the information required to create, for example, the PayPal equivalent of a Stripe Subscription via the PayPal API.

### IAP Store Customers

A Catena account may have one customer per IAP store created on their behalf.

When a user purchases a product, one of two actions will take place:

1. Their `external_customer_id` will be pulled from `entitlements.iap_store_customers` and used in purchase processing.
2. A new customer will be created for them with the IAP store within which they are making the purchase and their `external_customer_id` will be stored in `entitlements.iap_store_customers`.

*******Note: `entitlements.iap_store_customers` is a database table that maps `account_ids` to `external_customer_ids` & `iap_stores`.*

### Stripe Purchase Processing

When a user purchases a product from Stripe, a new Stripe Subscription will be created. If the `GrantEntitlementsBeforePaymentConfirmation` config variable is set to `true`, this subscription will have a status of `active` by default and the user will be emailed the subscription’s first invoice which they will have `DaysUntilFirstInvoicePaymentDue` days to pay.

Otherwise, the subscription will have a status of `incomplete` by default, until the first invoice for the subscription is paid, at which point the status is updated to `active`.

*Note: Catena creates a Stripe Subscription in both of the following cases:*

1. *A user makes a **“one-time purchase”.***
2. *A user makes a purchase for which they will be **billed on a recurring basis**.*

*The difference is that a “one-time purchase” will have only a single invoice, and invoices will be sent on a recurring basis for purchases such as monthly memberships.*

If the `GrantEntitlementsBeforePaymentConfirmation` config variable is set to `true`, entitlements associated with this purchase will be granted to the purchasing user regardless of whether or not the success of their first invoice payment has been validated yet.

Otherwise, associated entitlements will be granted on the next run of the `TransactionReceiptValidationJob` if the subscription has been activated since creation. Again, this status update would be triggered by a successful payment of the first invoice sent for this purchase.

Account entitlements are tracked in `entitlements.account_entitlements`:

```protobuf
// an account may have multiple granted entitlements
Create.Table("account_entitlements")
    .WithColumn("account_entitlement_id").AsString().PrimaryKey()
    .WithColumn("account_id").AsString()
    .WithColumn("entitlement_id").AsString()
        .ForeignKey("entitlements_account_entitlements", "entitlements", "entitlement_id");
```

The final step of processing a purchase from one of our supported IAP stores is storing a transaction receipt for the purchase in `entitlements.transactions`.

```protobuf
Create.Table("transactions")
    .WithColumn("transaction_id").AsString().PrimaryKey().Unique()
    .WithColumn("external_product_id").AsString()
    .WithColumn("iap_store").AsString()
    .WithColumn("receipt").AsString(); // serialized transaction receipt
```

This transaction receipt will be validated periodically (currently daily at midnight) by a cron job titled `TransactionReceiptValidationJob`.

## Transaction Receipt Validation - `TransactionReceiptValidationJob`

Transaction receipts are validated periodically (currently daily at midnight) by a cron job titled `TransactionReceiptValidationJob` leveraging various “transaction receipt validators” each configured to validate receipts from one IAP store.

Each of these will implement the `ITransactionReceiptValidator` interface, and provide an implementation for that interface’s `Validate()` function.

### Invalid Receipts

If the validation of a receipt fails due to an inability to connect to an IAP store’s API and the `RevokeEntitlementsOnFailedStripeAPIRequests` config variable is set to `true`, entitlements previously granted from this purchase will be revoked from the purchasing user’s account. Otherwise, the account will retain its entitlements.

The fail closed approach used when `RevokeEntitlementsOnFailedStripeAPIRequests` is set to `true` exists to support cases where clients wish to have a user’s entitlements revoked if we can not confirm that the most recent invoice for their subscription has been paid.

**Note: Entitlements will never be revoked from a user’s account due to a Catena application error.**

If a receipt is legitimately deemed invalid, entitlements previously granted from the purchase will be revoked from the purchasing user’s account.

### Valid Receipts

If a transaction receipt validation succeeds, the account of the user who made the purchase will be granted the purchase’s corresponding entitlements if they do not already posses them and retain them otherwise.

### The StripeTransactionReceiptValidator

Stripe Subscriptions with a `trialing` or an `active` status are deemed valid.

Those with an `incomplete` or a `past_due` status are deemed invalid.

*Note:*

*`incomplete` - the initial payment attempt fails,*

`*past_due` - a subsequent invoice is not paid by its due date*

Those with an `incomplete_expired`, a `canceled`, or an `unpaid` status are deemed invalid and are considered by Stripe to be in a “terminal state” meaning that a user will have to complete the purchase process for this product again to be granted the purchase’s corresponding entitlements again.

**Note:**

`incomplete_expired` - first invoice has not been paid within 23 hours of finalization,

`canceled` - a customer has cancelled their subscription or an invoice has not been paid by an additional deadline after its due date,

`unpaid` - an invoice has not been paid by an additional deadline after its due date