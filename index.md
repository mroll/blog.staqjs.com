# Subscription Billing with Stripe Customer Portal and Firebase Functions

**2020-08-15**

The Stripe [Customer Portal](https://stripe.com/docs/billing/subscriptions/customer-portal) is one of the coolest things ever to happen to independent SaaS. Before the Portal, there was a lot of logic and UI that you had to write custom code for to handle various subscription events like creation, cancellation, upgrade and downgrade. There was a lot of documentation you had to read before you could be sure you were writing the correct logic. And it was kind of important to read that documentation, since messing up payments can mean really bad things for your business. With the Customer Portal, all that stress of learning how to properly write subscription management goes away. In less than 100 lines of code, you can have a production-ready subscription billing service ready to power your passive SaaS income :)

If you want the power of managed subscription billing in your application ASAP, you might be interested in Staq, where those 100 lines of code have already been written for you. Integrating into your Firebase functions is as easy as this:

```js
// index.js
const { initStaq, createStripeCustomerPortalSession, stripeCustomer } = require('@staqjs/server')

initStaq({
  gcpProjectNumber: <your-gcp-project-number>,
  stripeTrialPriceId: <price-id-for-your-default-plan>,
})

exports.stripeCustomer = stripeCustomer
exports.createStripeCustomerPortalSession = createStripeCustomerPortalSession
```

If you're also interested in how it all works, please read on.

Unfortunately, the Customer Portal does not yet give users the ability to create subscriptions. So you have to write subscription creation logic yourself. Which would be kind of a pain if you needed to write a UI that lets users pick their plan and then explicitly sign up for a subscription. When I first discovered the Portal, that's what I thought I would have to do :(

But then I realized Stripe allows you to create a subscription for a user without them adding payment info. If you don't need to collect credit card info, you can just create a default subscription for every user when they sign up!

This technique can support the most widely used SaaS business models: freemium and limited-time free trial.

For a freemium model, use the Stripe dashboard to create a [Product](https://stripe.com/docs/billing/prices-guide) that has a recurring price of $0/month. This is the default plan that every user will be subscribed to when they sign up. In your product code, all users who are subscribed to this plan should be given access to the free tier features.

For a limited-time free trial model, use the Stripe dashboard to create a Product that has a non-zero recurring price. Then in your Subscription creation code, set the `trial_period_days` field on the Subscription object that you submit to Stripe. Stripe will treat this Subscription as "active" for the number of days you specify, and then it will attempt to charge the associated Customer. If the Customer has added payment info and the payment is successful, Stripe will go on treating this Subscription as active. If the user has not added their payment info or the payment fails for another reason, Stripe will deactivate the Subscription.

Here is the server-side code Staq uses for simultaneously creating a Customer and a Subscription when a new user signs up. It supports both the freemium and limited-time trial setups described above.

```js
async function createCustomer(data, context, stripe) {
  try {
    const customer = await stripe.customers.create(data.customer)
    const subscriptionObj = {
      customer: customer.id,
      items: [
        { price: staqConfig.get('stripeDefaultPriceId') }
      ],
    }

    if (staqConfig.get('useTrial')) {
      subscriptionObj.trial_period_days = staqConfig.get('stripeTrialPeriodDays') || 14
    }

    const subscription = await stripe.subscriptions.create(subscriptionObj)

    return {
      customer,
      subscription,
    }
  } catch (error) {
    console.error(error)
    return {
      error,
    }
  }
}
```

The beauty of creating a Subscription for every new user on signup is that by doing so you can outsource subscription management to Stripe via the Customer Portal. Just give users a button somewhere in your product that links them to the Portal. This is what it looks like in the default Staq template.

![billing-details](/assets/billing-details.png)

Now, for security reasons Stripe doesn't permanently host a Customer Portal page for every user. You wouldn't want just anyone to be able to access it if they got hold of the link. The way Portal access works is Stripe assumes you have properly authenticated a user through your application's login mechanism, and it will generate a short-lived access token for a given user when requested. The "Manage Billing" button you provide should make that request and then direct the user to the fresh Customer Portal session. Here's what that request looks like in Staq, both the client code that gets triggered when clicking the button, and the server-side code that uses the Stripe API to request an access token.

```js
// client code
const onClickManageBilling = () => {
  const createPortalSession = firebase.functions.httpsCallable('createStripeCustomerPortalSession')
  createPortalSession({
    customerId: auth.currentUser.stripeCustomerId,
    return_url: staqConfig.get('urlBase'),
  }).then((result) => {
    window.location = result.data.url
  }).catch((error) => {
    // error handling
  })
}
```

```js
// server code
async function createPortalSession(data, context, stripe) {
  const session = await stripe.billingPortal.sessions.create({
    customer: data.customerId,
    return_url: data.return_url
  })

  return session
}
```

And with that, you've successfully outsourced subscription management to the experts. This will hopefully make a big impact on your business for two reasons:

1. You will be able to deliver your product to users and start getting paid without spending weeks building subscription management
2. You will not stress about messing up payments and will generally be a happier person

**Notes**

* One thing this post does not cover is [subscription webhooks](https://stripe.com/docs/billing/subscriptions/webhooks). Depending on how you store data about user subscriptions, it can be necessary to implement logic for handling events that Stripe will send you. Staq has yet to implement that logic, but be sure that it's coming soon! When it does, we will post a detailed guide about how to think about webhooks in the context of Customer Portal and Firebase functions, and how you can easily integrate them using Staq.


<br/>
<br/>
<br/>

# Introducing Staq

**2020-08-09**

In the first half of 2020, I built three web applications. They all used the same stack: React and Firebase. The combination of these two tools is the fastest way I know how to build out prototypes for web app ideas.

But I noticed something frustrating while building these three projects. I had to write a bunch of the same code for each one. In fact, I found myself straight up copying files from the first project to the other two. And the code I was copying was code that has to exist in one form or another in just about every web application out there: components and logic for handling users, helper functions for interacting with the database, backend logic for managing subscription billing, and standard components like the landing page and footer.

So I decided to create Staq. An open source library I can call into whenever I need standard SaaS building blocks.

I initially toyed with the idea of charging for Staq as a code generator. But I discovered I care a lot about the potential this tool has for unlocking productivity on the web, and I don't want to limit the userbase to people who can justify paying to use it.

When I look at my experience building out product ideas, the boring and time consuming part is the work necessary to hook together these amazing services like Stripe and Firebase that make it possible to run a successful business from your couch with almost no startup capital. I think this type of boring plumbing is a significant barrier to entry that stops a lot of potentially great and profitable businesses from getting started. And just like the role Stripe and Firebase play in payments and hosting, I want there to be a product that lets you get over that barrier quickly and without upfront payment. As an open source project that is available as a free npm package, I think Staq can fill that role.

The project is super early and has a long way to go, but the proof concept is working. You can follow the [Getting Started](https://staqjs.com/gettings-started/) guide on the documentation site and have a SaaS app up and running in minutes, complete with user auth and subscription billing. I'm even using Staq to power a side project of mine, [checkfox.app](https://checkfox.app).
