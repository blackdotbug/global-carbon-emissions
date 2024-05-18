<script>
	import { geoOrthographic, geoPath } from 'd3-geo';
	import { drag } from 'd3-drag';
	import { json } from 'd3-fetch';
	import { select } from 'd3-selection';
    import { extent, range } from 'd3-array';
    import { scaleQuantize, scaleLinear } from 'd3-scale';
    import { schemeOranges } from 'd3-scale-chromatic';
	import { onMount } from 'svelte';
    import { feature, mesh } from 'topojson-client';
    import Glasses from './Glasses.svelte';

    export let emissions;

	// Map setup & rendering
	let projection = geoOrthographic().translate([480,480]);
	let path = geoPath().projection(projection);
	let rotation = [0, 0, 0]; // Initial rotation
	let sphere = { type: 'Sphere' }; // Globe Outline
	let land, borders, countries, selected, selectedData, glassNodes, size;
    let colorrange = extent(emissions.map(m => m.emissions));
    let colorscale = scaleQuantize(colorrange, schemeOranges[9]);
    let valuemap = new Map(emissions.map(m => [m.name, m.emissions]));
    let radiusRange = extent(emissions.map(m => m.emissions));
    let radiusScale = scaleLinear(radiusRange, [10, 50]);
	// Reactive code to update on map dragging
	$: if (projection) {
		projection.rotate(rotation);
		path = geoPath().projection(projection);
	}
    $: selectedData = emissions.find(f => f.name == selected?.properties.name);
    $: selectedSize = radiusScale(selectedData?.emissions);

    /**
	 * Calculates the rotation of the globe when the user drags on the map
	 *
	 * @param event
	 */
	function dragged(event) {
		const dx = event.dx;
		const dy = event.dy;
		const currentRotation = projection.rotate();
		const radius = projection.scale();
		const scale = 360 / (2 * Math.PI * radius);

		rotation = [
			currentRotation[0] + dx * scale,
			currentRotation[1] - dy * scale,
			currentRotation[2]
		];

		projection.rotate(rotation);
	}
    
	onMount(async () => {
		// Geo Data from World-Atlas
		const world = await json('https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json');
		land = feature(world, world.objects.land);
		borders = mesh(world, world.objects.countries, (a, b) => a !== b);
        countries = feature(world, world.objects.countries).features;

		const globe = select('.globe-path');

		// Define drag behavior
		const dragHandler = drag().on('drag', (event) => {
			dragged({ dx: event.dx, dy: event.dy });
		});

		// Apply the drag behavior
		dragHandler(globe);
	});
    function handleClick(e, country) {
        selected = country;
        if (selectedData) {
            let glassNumber = Math.ceil(selectedData.population/100000);
            glassNodes = range(glassNumber);
        }
    }
</script>

<svg width="100%" height="100%" viewBox="0 0 960 960" preserveAspectRatio="xMidYMid meet">
    <g>
        <!-- Globe background -->
        <path d={path(sphere)} stroke="none" fill="rgba(73, 194, 227)" />

        <!-- Globe outline with transparent fill to make it completly draggable -->
        <path d={path(sphere)} fill="rgba(0,0,0,0)" stroke="#000" class="globe-path" />
    
        <!-- Land Outline -->
        <path d={path(land)} fill="none" stroke="#000" />
    
        <!-- Countries Filled -->
        {#if countries}
        {#each countries as country}
            <path
                d={path(country)}
                stroke="none"
                fill={colorscale(valuemap.get(country.properties.name))}
                on:click={(e) => handleClick(e,country)}
                on:keypress={() => selected = country}
            />
        {/each}
        {/if}
    
        <!--Countries' Borders -->
        <path d={path(borders)} fill="none" stroke="#000" />    
    </g>
    {#if selected && glassNodes}
        <Glasses {glassNodes} size={selectedSize} />
    {/if}
</svg>
