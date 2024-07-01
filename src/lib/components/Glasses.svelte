<script>
    import Magnify from './Magnify.svelte';
    import { forceSimulation, forceCollide, forceRadial } from 'd3-force';

    export let glassNodes;
    export let size;
    export let width;

    $: initialNodes = glassNodes.map((d)=>({ x: width/2, y: 0, ...d }));
    $: simulation = forceSimulation(initialNodes);
    $: nodes = [];
    $: {
        simulation
            .force("charge", forceCollide().radius(10))
            .force("r", forceRadial(width/2, width/2, width/2).strength(0.9))
            .alphaTarget(0.00001)
            .on("tick", () => {
                nodes = simulation.nodes()
            }).restart();
    }

</script>
<div class="flex items-center justify-center h-screen pointer-events-none">
    <div class="mx-auto my-auto" style="width:{Math.ceil(width)}px;height:{Math.ceil(width)}px">
        {#each nodes as glass}
            <Magnify x={glass.x} y={glass.y} {size} class="absolute"/>
        {/each}
    </div>    
</div>
