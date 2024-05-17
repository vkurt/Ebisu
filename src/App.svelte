<script>
  import svelteLogo from './assets/svelte.svg'
  import viteLogo from '/vite.svg'
  import Counter from './lib/Counter.svelte'
  import Router from "svelte-spa-router"
  import Balances from "./router/Balances.svelte"
  import {
    Navbar,
    NavBrand,
    NavLi,
    NavUl,
    NavHamburger,
    ImagePlaceholder,
    Skeleton,
    TextPlaceholder,
  } from "flowbite-svelte";
  import { DarkMode } from "flowbite-svelte";
  import Pools from "./router/Pools.svelte";
  import SendToken from './router/SendToken.svelte'
</script>

<Navbar >
    <NavBrand href="/">
      <img src="/images/flowbite-svelte-icon-logo.svg" class="me-3 h-6 sm:h-9" alt="Flowbite Logo" />
      <span class="self-center whitespace-nowrap text-xl font-semibold dark:text-white">Flowbite</span>
    </NavBrand>
    <NavHamburger  />
    <NavUl >     
      <NavLi href="/#/pools">Pools</NavLi>
      <NavLi href="/#/">Balances</NavLi> 
      <NavLi href="/#/sendtoken">Send Token</NavLi>
    </NavUl>
    <DarkMode />
</Navbar>


  
<Router
  routes={{

    "/": Balances,
    "/pools": Pools,
    "/sendtoken": SendToken
   
  }}>
  </Router>




