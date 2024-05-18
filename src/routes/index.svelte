<script lang="ts">
  import MainFullVh from "$lib/components/MainFullVh.svelte"
  import Globe from "$lib/components/Globe.svelte";
  import Hero from "$lib/components/Hero.svelte";
  import Footer from "$lib/components/Footer.svelte";
  import emissions from "$lib/data/globalCO2.json";
</script>

<MainFullVh>
  <Hero/>
  <Globe {emissions}/>
  <Footer/>
</MainFullVh>
