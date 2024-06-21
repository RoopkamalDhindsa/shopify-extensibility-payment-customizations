# Shopify extensibility payment customization App Template - Remix

This is a template for building a [Shopify app](https://shopify.dev/docs/apps/getting-started) using the [Remix](https://remix.run) framework.

<!-- TODO: Uncomment this after we've started using the template in the CLI -->
<!-- Rather than cloning this repo, you can use your preferred package manager and the Shopify CLI with [these steps](#installing-the-template). -->

## Step 1: Setup Shopify app and install it on dev store

### Prerequisites

1. You must [download and install Node.js](https://nodejs.org/en/download/) if you don't already have it.
2. You must [create a Shopify partner account](https://partners.shopify.com/signup) if you don’t have one.
3. You must create a store for testing if you don't have one, either a [development store](https://help.shopify.com/en/partners/dashboard/development-stores#create-a-development-store) or a [Shopify Plus sandbox store](https://help.shopify.com/en/partners/dashboard/managing-stores/plus-sandbox-store).

<!-- TODO Make this section about using @shopify/app once it's added to the CLI. -->

### Setup

If you used the CLI to create the template, you can skip this section.

1. Run the below command to install the node module.

```shell
npm install
```

2. If you received an error about fixing the npm audit then run the below commands as required:

```shell
npm audit fix
```
OR

```shell
npm audit fix --force
```

### Local Development

```shell
npm run dev
```

Press P to open the URL to your app. Once you click install, you can start development.



## Step 2: Create the delivery customization function

### 1. Navigate to `extensions/payment-customization-js`
Press Ctrl+C to stop the app development environment and then, use cd to go to the exrtension's main folder

```shell
cd extensions/payment-customization-js
```

### 2. Replace the contents of `src/run.graphql` file with the following code.
`run.graphql` defines the input for the function. You need the cart delivery groups, with the delivery state/province code and available delivery options.

```shell
query RunInput {
  cart {
    cost {
      totalAmount {
        amount
      }
    }
  }
  paymentMethods {
    id
    name
  }
}
```

### 3. Run the following command to regenerate types based on your input query

```shell
npm run shopify app function typegen
```

### 4. Replace the `src/run.js` file with the following code.
This function logic appends a message to all delivery options if the shipping address state or province code is NC. You can adjust this to the state or province of your choice.

```shell
// @ts-check

// Use JSDoc annotations for type safety
/**
* @typedef {import("../generated/api").RunInput} RunInput
* @typedef {import("../generated/api").FunctionRunResult} FunctionRunResult
*/

/**
* @type {FunctionRunResult}
*/
const NO_CHANGES = {
  operations: [],
};

// The configured entrypoint for the 'purchase.payment-customization.run' extension target
/**
* @param {RunInput} input
* @returns {FunctionRunResult}
*/
export function run(input) {
  // Get the cart total from the function input, and return early if it's below 100
  const cartTotal = parseFloat(input.cart.cost.totalAmount.amount ?? "0.0");
  if (cartTotal < 100) {
    // You can use STDERR for debug logs in your function
    console.error("Cart total is not high enough, no need to hide the payment method.");
    return NO_CHANGES;
  }

  // Find the payment method to hide
  const hidePaymentMethod = input.paymentMethods
    .find(method => method.name.includes("Cash on Delivery"));

  if (!hidePaymentMethod) {
    return NO_CHANGES;
  }

  // The @shopify/shopify_function package applies JSON.stringify() to your function result
  // and writes it to STDOUT
  return {
    operations: [{
      hide: {
        paymentMethodId: hidePaymentMethod.id
      }
    }]
  };
};
```

## Step 3: Preview the function on a development store

### 1. Navigate back to your app root:

```shell
cd ../..
```

### 2. Use the Shopify CLI dev command to start app preview:

```shell
npm run shopify app dev
```

## Step 4: Create the delivery customization with GraphiQL

### 1. Install the Shopify [GraphiQL app][(https://shopify-graphiql-app.shopifycloud.com/login) on your store. If you've already installed GraphiQL, then you should do so again to select the necessary access scopes for delivery customizations.

### 2. In the GraphiQL app, in the API Version field, select the 2023-07 version.

### 3. Find the ID of your function by executing the following query:

```shell
query {
  shopifyFunctions(first: 25) {
    nodes {
      app {
        title
      }
      apiType
      title
      id
    }
  }
}
```
The result contains a node with your function's ID:

```shell
{
  "app": {
    "title": "your-app-name-here"
  },
  "apiType": "payment_customization",
  "title": "payment-customization",
  "id": "YOUR_FUNCTION_ID_HERE"
}
```

### 4. Execute the following mutation and replace `YOUR_FUNCTION_ID_HERE` with the ID of your function:

```shell
mutation {
  paymentCustomizationCreate(paymentCustomization: {
    title: "Hide payment method by cart total",
    enabled: true,
    functionId: "YOUR_FUNCTION_ID_HERE",
  }) {
    paymentCustomization {
      id
    }
    userErrors {
      message
    }
  }
}
```
You should receive a GraphQL response that includes the ID of the created delivery customization. If the response includes any messages under `userErrors`, then review the errors, check that your mutation and `functionId` are correct, and try the request again.

Tip*
If you receive a Could not find Function error, then confirm the following:
- The function ID is correct.
- You've installed the app on your development store.
- Development store preview is enabled.


## Step 5: Test the delivery customization

### 1. From the Shopify admin, go to `Settings > Payments`.

### 2. Check the `Payment customizations` section. You should find the `Hide payment method by cart total` payment customization that you created with GraphiQL.

### 3. From the `Manual payment methods` section, click `Add manual payment method` and then click `Cash on Delivery (COD)`.

### 4. Click `Activate`.

### 5. Open your development store and build a cart with a total (including shipping and tax) under 100. The `Cash on Delivery` payment method should display in checkout.

### 6. Add additional items to your cart to raise the total over 100. Your payment function should now hide the `Cash on Delivery` payment option.

### 7. To [debug your function](https://shopify.dev/docs/apps/build/functions/monitoring-and-errors#debug-a-function), or view its output, you can review its logs in your [Partner Dashboard](https://partners.shopify.com/organizations).

# Next steps
- [Add configuration](https://shopify.dev/docs/apps/build/checkout/payments/add-configuration) to your payment customization using metafields.
