<script>
  import { onMount } from 'svelte';
  import Page from './routes/+page.svelte';
</script>

<main>
  <h1>SURIMP</h1>
  
  <p class="description">
   SURIMPRESSION PHOTOGRAPHIQUE
  </p>

  <Page />
</main>

<style>
  
  main {
    text-align: center;
    padding: 1em;
    margin: 0 auto;
  }

  h1 {
    color: #ff3e00;
    text-transform: uppercase;
    font-size: 2em;
    font-weight: 100;
    margin-bottom: 0.5em;
  }

  .description {
    color: #666;
    margin-bottom: 2em;
  }

  @media (min-width: 640px) {
    main {
      max-width: 1920px;
    }
  }
</style>
