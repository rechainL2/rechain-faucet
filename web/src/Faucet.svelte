<script>
  import { onMount } from 'svelte';
  import { getAddress } from '@ethersproject/address';
  import { CloudflareProvider } from '@ethersproject/providers';
  import { setDefaults as setToast, toast } from 'bulma-toast';

  let input = null;
  let faucetInfo = {
    account: '0x0000000000000000000000000000000000000000',
    network: 'testnet',
    payout: 1,
    symbol: 'ETH',
    hcaptcha_sitekey: '',
  };

  let mounted = false;
  let hcaptchaLoaded = false;

  onMount(async () => {
    const res = await fetch('/api/info');
    faucetInfo = await res.json();
    mounted = true;
  });

  window.hcaptchaOnLoad = () => {
    hcaptchaLoaded = true;
  };

  $: document.title = `Re:Chain Faucet`;

  let widgetID;
  $: if (mounted && hcaptchaLoaded) {
    widgetID = window.hcaptcha.render('hcaptcha', {
      sitekey: faucetInfo.hcaptcha_sitekey,
    });
  }

  setToast({
    position: 'bottom-center',
    dismissible: true,
    pauseOnHover: true,
    closeOnClick: false,
    animate: { in: 'fadeIn', out: 'fadeOut' },
  });

  async function handleRequest() {
    let address = input;
    if (address === null) {
      toast({ message: 'input required', type: 'is-warning' });
      return;
    }

    if (address.endsWith('.eth')) {
      try {
        const provider = new CloudflareProvider();
        address = await provider.resolveName(address);
        if (!address) {
          toast({ message: 'invalid ENS name', type: 'is-warning' });
          return;
        }
      } catch (error) {
        toast({ message: error.reason, type: 'is-warning' });
        return;
      }
    }

    try {
      address = getAddress(address);
    } catch (error) {
      toast({ message: error.reason, type: 'is-warning' });
      return;
    }

    try {
      let headers = {
        'Content-Type': 'application/json',
      };

      if (hcaptchaLoaded) {
        const { response } = await window.hcaptcha.execute(widgetID, {
          async: true,
        });
        headers['h-captcha-response'] = response;
      }

      const res = await fetch('/api/claim', {
        method: 'POST',
        headers,
        body: JSON.stringify({
          address,
        }),
      });

      let { msg } = await res.json();
      let type = res.ok ? 'is-success' : 'is-warning';
      toast({ message: msg, type });
    } catch (err) {
      console.error(err);
    }
  }

  function capitalize(str) {
    const lower = str.toLowerCase();
    return str.charAt(0).toUpperCase() + lower.slice(1);
  }
</script>

<svelte:head>
  {#if mounted && faucetInfo.hcaptcha_sitekey}
    <script
      src="https://hcaptcha.com/1/api.js?onload=hcaptchaOnLoad&render=explicit"
      async
      defer
    ></script>
  {/if}
</svelte:head>

<main class="main">
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Roboto+Serif:ital,opsz,wght@0,8..144,100..900;1,8..144,100..900&display=swap');
  </style>

  <div class="container">
    <div class="header">
      <img class="header__logo" src="/favicon.png" alt="">
      <h1 class="header__heading">Re:Chain Faucet</h1>
      <p class="header__subheading">Lorem ipsum dolor sit amet consectetur adipisicing elit. Voluptatem similique repellendus ab iste magni. Voluptates nulla qui culpa dolore assumenda?</p>
    </div>

    <div class="form">
      <p class="form__address">
        <strong>Serving from</strong>
        <code>{faucetInfo.account}</code>
      </p>

      <input
        class="form__input"
        bind:value={input}
        type="text"
        placeholder="Enter your address or ENS name"
      />

      <button
        class="form__button"
        on:click={handleRequest}
      >
        Request {faucetInfo.payout} Re:Chain Token
      </button>
    </div>
  </div>

  <img class="footer-image" src="/footer-bg.png" alt="">
</main>

<style>
  * {
    box-sizing: border-box;
    font-family: 'Roboto Serif', sans-serif;
  }

  .main {
    display: flex;
    flex-direction: column;
    min-height: 100vh;
  }

  .container {
    width: 100%;
    max-width: 520px !important;
    margin-left: auto;
    margin-right: auto;
    padding-left: 16px;
    padding-right: 16px;
  }

  .header {
    padding-top: 6rem;
    text-align: center;
  }

  .header__logo {
    display: block;
    width: 120px;
    height: 120px;
    margin: 0 auto 16px;
  }

  .header__heading {
    font-size: 24px;
    font-weight: 500;
    color: #000;
    margin-bottom: 8px;
  }

  .form {
    margin-top: 32px;
    border: 1px solid rgb(222, 226, 230);
    border-radius: 16px;
    padding: 24px;
    margin-bottom: 5rem;
  }

  .form__address {
    display: flex;
    flex-direction: column;
    align-items: center;
    font-size: 14px;
    color: #000;
  }

  .form__address > strong {
    font-weight: 400;
    font-size: 18px;
    line-height: 24px;
  }

  .form__address > code {
    font-size: 18px;
    color: #666;
    background: transparent !important;
  }

  .form__input {
    width: 100%;
    padding: 8px 12px;
    margin-top: 16px;
    border: 1px solid rgb(222, 226, 230);
    border-radius: 12px;
    font-size: 16px;
    line-height: 30px;
    font-family: monospace;
  }

  .form__button {
    width: 100%;
    padding: 12px;
    margin-top: 16px;
    border: none;
    border-radius: 32px;
    font-size: 16px;
    line-height: 24px;
    font-weight: 500;
    color: #fff;
    background-color: #40c057;
    cursor: pointer;
    font-weight: 600;
  }

  .footer-image {
    flex: none;
    max-height: 160px;
    margin: 24px auto 0;
  }
</style>
