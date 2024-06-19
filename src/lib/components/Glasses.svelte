<script>
    import Magnify from './Magnify.svelte';
    import { forceSimulation, forceCollide, forceRadial } from 'd3-force';

    export let glassNodes;
    export let size;

    $: initialNodes = glassNodes.map((d)=>({ ...d }));
    $: simulation = forceSimulation(initialNodes);
    $: nodes = [];
    $: {
        simulation
            .force("charge", forceCollide().radius(10).iterations(4))
            .force("r", forceRadial(400, 480, 480).strength(0.9))
            .alphaTarget(0.001)
            .on("tick", () => {
                nodes = simulation.nodes()
            }).restart();
    }

</script>
<g>
    {#each nodes as glass}
        <Magnify x={glass.x} y={glass.y} {size}/>
    {/each}
</g>
