# strip_payment_nextJS

Step 01: Nextjs_Stripe_Integration

    First create Nextjs 13 app and add product page, you can see in /components/Product/Product.tsx
    Create stripe account and open dashboard then add new a project
    Install stripe in you local project

npm install stripe

    Add the required environment variables. Create a new file .env.local in the root directory with the following data.

NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=YOUR_STRIPE_PUBLISHABLE_KEY
STRIPE_SECRET_KEY=YOUR_STRIPE_SECRET_KEY

    You can get these credentials from Dashboard -> Developers -> API Keys.
    Now need to build an API to get the session id that is required for redirecting the user to the checkout page.
    Create a new file in api/create-stripe-session.ts. And add the following.

import { NextResponse } from "next/server";
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY as string, {
  apiVersion: "2022-11-15",
});

    Creating the shape for the item needed by Stripe.

There is a particular type of object which Stripe expects to get, this is the object. You should use your local currency instead of "usd" if you want.

const transformedItem = {
         price_data: {
          currency: 'usd',
          product_data:{
            name: item.name,
            description: item.description,
            images:[item.image],
            metadata:{name:"some additional info",
                     task:"Usm created a task"},

          },
          unit_amount: item.price * 100,

        },
        quantity: item.quantity,
        
      };

Creating Stripe Session in the backend

in api/create-stripe-session.ts. file you will need to create a stripe session object where you need to define some data.

 const redirectURL =
    process.env.NODE_ENV === 'development'
      ? 'http://localhost:3000'
      : 'your live vercel app link';

      const session = await stripe.checkout.sessions.create({
        payment_method_types: ['card'],
        line_items: [transformedItem],
        mode: 'payment',
        success_url: redirectURL + '/payment/success',
        cancel_url: redirectURL + '/payment/fail',
        metadata: {
          images: item.image,
        },
      });
      return NextResponse.json(session?.id) ;

    payment_method_type: In this, we add the payment methods to pay the price of the product. Click here to know more payment methods.
    success_url: In success_url, you define where the user will go after the payment is successful.
    cancel_url: In the cancel_url, you define where the user will go if the user clicks the back button. It can be a cancel page or the checkout page as well.
    metadata: In metadata, we will add images of the product, if you want you can add other options too.

Now, our backend is ready, now we have to send a POST request to API to get the session.
Redirecting to Stripe Checkout Page

install following library

import { loadStripe } from "@stripe/stripe-js";

npm install @stripe/stripe-js

in your Product.tsx file add below code

  const publishableKey = process.env
    .NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY as string;
  const stripePromise = loadStripe(publishableKey);

    Now, we'll create createCheckoutSession function to get the Stripe Session for the checkout.

const createCheckOutSession = async () => {
    
    const stripe = await stripePromise;

    const checkoutSession = await fetch(
      "http://localhost:3000/api/create-stripe-session",
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          item: item,
        }),
      }
    );

    const sessionID= await checkoutSession.json();
    const result = await stripe?.redirectToCheckout({
      sessionId: sessionID,
    });
    if (result?.error) {
      alert(result.error.message);
    }
  };

Now, we have to call this function while the user clicks the Buy button. And onClick={createCheckoutSession}

<button
  disabled={item.quantity === 0}
  onClick={createCheckOutSession}
  className='bg-blue-500 hover:bg-blue-600 text-white block w-full py-2 rounded mt-2 disabled:cursor-not-allowed disabled:bg-blue-100'
>
  Buy
</button>

    we have came to end now run your project

npm run dev

Step 02: Webhooks setup
what is webhook?

A webhook is a method of communication used by applications or services to send real-time data or notifications to other applications or services. It is a way for one application to provide information to another application automatically and instantly Webhooks can be used to send real-time notifications about specific events, such as new user registrations, payment confirmations, or status updates

Adding following step to integrate stripe webhooks -First thing in your project add api/webhook/route.ts endpoint which will handle the calls coming from stripe

    Add POST method and paste the following code

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY as string, {
  apiVersion: "2022-11-15",
});
const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET as string;

export  async function POST(req: any, res: any){
    
    try {
    const rawBody = await req.text();
    const sig = req.headers.get("stripe-signature") as string
    const event = stripe.webhooks.constructEvent(rawBody,sig,webhookSecret);
    if ( 'checkout.session.completed' === event.type ) {
        const session = event.data.object;
       //Once you'll get data you can use it according to your 
        //reqirement for making update in DB
    } else {
        res.setHeader("Allow", "POST");
        // res.status(405).end("Method Not Allowed");
    }
    } catch (err: any) {
        console.log("Error in webhook----------", err);
        // res.status(400).send(`Webhook Error: ${err.message}`);
        return;
    }
   
}

Note: you'll get your webhookSecret from stripe dashboard

    After adding above step now go to your stripe dashboard in developers mode open webhooks tab or you simple search webhooks in search bar
    You'll find two parts "Hosted endpoints' and "Local listeners"
    Click on Add endpoint button
    New page will open simple enter url of your hosted project webhook api
    Select events to listen to checkout.session.completed and click on Add endpoint button

Above stepup is for deployed project if you want to test your webhook locally then you need to do some extra work
Test in local Enviroment

    Download stripe-cli from here according to your machine Operation system
    You'll get zip file, extract that file and open cmd in extracted directory
    Run following command in cmd

stripe login

    After login forward events to your webhook run followind command

stripe listen --events checkout.session.completed --forward-to localhost:3000/api/webhook

    Once you run above command you'll get Your webhook signing secret which will look like this "whsec_08d........"
    copy your webhook secret key and paste in your project .env file with name of STRIPE_WEBHOOK_SECRET

Now run your local project do checkout and once payment flow completed your local webhook api will hit and you'll get data about the transaction and you can use the coming data according to your need
