<script>
	import { geoOrthographic, geoPath, geoCentroid } from 'd3-geo';
	import { drag } from 'd3-drag';
	import { json } from 'd3-fetch';
	import { select } from 'd3-selection';
    import { extent, range } from 'd3-array';
    import { scaleQuantize, scaleLinear } from 'd3-scale';
    import { format } from 'd3-format';
    import { schemeOranges } from 'd3-scale-chromatic';
	import { onMount } from 'svelte';
    import { feature, mesh } from 'topojson-client';
    import Glasses from './Glasses.svelte';

    export let emissions;

	// Map setup & rendering
	let rotation = [0, 0, 0]; // Initial rotation
	let sphere = { type: 'Sphere' }; // Globe Outline
	let land, borders, countries, selected, glassNodes, globe, baseSVG, baseRect, projection, path;
    let colorrange = extent(emissions.map(m => m.emissions));
    let colorscale = scaleQuantize(colorrange, schemeOranges[9]);
    let valuemap = new Map(emissions.map(m => [m.name, m.emissions]));
    let radiusRange = extent(emissions.map(m => m.emissions));
    let radiusScale = scaleLinear(radiusRange, [10, 50]);
    $: globeRect = getGlobeRect(globe);
    $: svgRect = getSvgRect();
	$: if (svgRect) {
        projection = geoOrthographic().fitExtent([[svgRect.left,svgRect.top+150],[svgRect.right,svgRect.bottom-150]], sphere);
    } else if (baseRect) {
        projection = geoOrthographic().fitExtent([[baseRect.left,baseRect.top+150],[baseRect.right,baseRect.bottom-150]], sphere);

    }
	$: if (projection) {
        path = geoPath().projection(projection);
    }
	// Reactive code to update on map dragging
	$: if (projection) {
		projection.rotate(rotation);
		path = geoPath().projection(projection);
	}
    $: selectedData = emissions.find(f => f.name == selected);
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
    function getSvgRect() {
        if (baseSVG) {
            return baseSVG.node().getBoundingClientRect();
        } else {
            return undefined;
        }
    }    
    function getGlobeRect(globe) {
        if (globe) {
            return globe.node().getBoundingClientRect();
        } else {
            return undefined;
        }
    }    

	onMount(async () => {
		// Geo Data from World-Atlas
		const world = await json('https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json');
		land = feature(world, world.objects.land);
		borders = mesh(world, world.objects.countries, (a, b) => a !== b);
        countries = feature(world, world.objects.countries).features;

		// Define drag behavior
		const dragHandler = drag().on('drag', (event) => {
			dragged({ dx: event.dx, dy: event.dy });
		});

        globe = select('.globe-path');
        baseSVG = select(".globe-svg");

        // Apply the drag behavior
		dragHandler(globe);

	});
    function handleClick(e) {
        selected = this.dataset.geo;
        let thisData = emissions.find(f => f.name == this.dataset.geo)
        let selectedCentroid = geoCentroid(countries.find(f => f.properties.name == this.dataset.geo));
        rotation = [-selectedCentroid[0], -selectedCentroid[1], rotation[2]];
        projection.rotate(rotation);
        if (thisData) {
            let glassNumber = Math.ceil(thisData.population/100000);
            let startGlass = range(glassNumber);
            glassNodes = startGlass.map(d => {
                return {x: 0, y: 0}
            })
        }
    }
</script>

<div class="w-full h-full relative" bind:contentRect={baseRect}>
<svg width="100%" height="100%" viewBox="0 0 {960} {960}" preserveAspectRatio="xMidYMid meet" class="absolute globe-svg" >
    {#if typeof path == "function"}
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
                data-geo={country.properties.name}
                d={path(country)}
                stroke="none"
                fill={colorscale(valuemap.get(country.properties.name))}
                on:click={handleClick}
                on:keypress={() => selected = country}
            />
        {/each}
        {/if}
    
        <!--Countries' Borders -->
        <path d={path(borders)} fill="none" stroke="#000" />    
    </g>
    {/if}
</svg>
{#if selectedSize && glassNodes && globeRect}
    <Glasses {glassNodes} size={selectedSize} width={globeRect.width} />
{/if}
{#if selected && selectedData}
    <div class="absolute top-10 left-10 text-white tracking-wide pointer-events-none">
        <h2 class="font-extrabold text-xl">{selected}</h2>
        <p class="font-thin">{selectedData.emissions} per capita emissions tonnes per person</p>
        <p class="font-thin">{format(",")(selectedData.population)} people</p>
    </div>
{/if}
</div>
