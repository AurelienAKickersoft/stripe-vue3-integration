<template>
  <form @submit.prevent="handleSubmit" class="fhr-bg-white fhr-shadow-md fhr-rounded fhr-px-8 fhr-pt-6 fhr-pb-8 fhr-mb-4">
    <div id="payment-element" class="fhr-mb-4">
      <!-- Stripe Payment Element will be inserted here. -->
    </div>
    <div class="fhr-flex fhr-items-center fhr-justify-between">
      <button class="fhr-bg-blue-500 hover:fhr-bg-blue-700 fhr-text-white fhr-font-bold fhr-py-2 fhr-px-4 fhr-rounded" :disabled="!stripe">
        Submit
      </button>
    </div>
  </form>
</template>

<script>
import { ref, onMounted } from 'vue';
import { loadStripe } from '@stripe/stripe-js';

export default {
  setup() {
    const stripePromise = loadStripe(process.env.STRIPE_PUBLIC_KEY);
    const stripe = ref(null);
    const elements = ref(null);
    const paymentElement = ref(null);
    let secret = ref(null)
    let method = ref(null)
    onMounted(async () => {
      stripe.value = await stripePromise;
      console.log('stripe', stripe)
      elements.value = stripe.value.elements();
      paymentElement.value = elements.value.create('card' );
      paymentElement.value.mount('#payment-element');

      const {clientSecret, error} = await fetch('http://127.0.0.1:80/payment/intent', {
        method: 'POST',
        headers:{
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({"total": 200})
      }).then(res => res.json());
      console.log(clientSecret, error);
      secret = clientSecret
    });

    const handleSubmit = async (event) => {
      if (!stripe.value || !elements.value) {
        // Stripe.js has not yet loaded.
        // Make sure to disable form submission until Stripe.js has loaded.
        return;
      }
      const data = await stripe.value.createPaymentMethod({
        type: 'card',
        card: paymentElement.value
      })
      console.log('method',method, data)

      const result = await stripe.value.confirmPayment( {
        clientSecret: secret,
        confirmParams: {
          payment_method: data.paymentMethod.id,
          return_url: 'http://localhost:5173/about'
        }
      });

      if (result.error) {
        // Show error to your customer
        console.log(result.error.message);
      } else {
        // The payment succeeded!
        console.log('succeed', result.paymentIntent);
      }
    };

    return {
      handleSubmit,
      stripe
    };
  }
}
</script>
