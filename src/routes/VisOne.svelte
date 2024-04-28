<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from "d3";
    import Page from "./+page.svelte";
    mapboxgl.accessToken =
        "pk.eyJ1Ijoianlvb25zb25nIiwiYSI6ImNsdW95emJzMzIxMDQya3FwcXc1NzA4c2sifQ.C9_2pEU8Z00WnIWNndTg_Q";

    let map;
    let mapViewChanged = 0;

    let properties = [];
    let radiusScale;
    onMount(async () => {
        map = new mapboxgl.Map({
            container: "mapone",
            style: "mapbox://styles/jyoonsong/cluq04ktu05cr01qqb4rp4v4f",
            center: [-71.09415, 42.36027],
            zoom: 12,
        });
        await new Promise((resolve) => map.on("load", resolve));

        properties = await d3.csv(
            "https://raw.githubusercontent.com/boston-price-dynamics/fp2/main/assess_lat_long.csv",
        );

        let totalUnits = d3.rollup(
            properties,
            (v) => v.length,
            (d) => d.GIS_ID,
        );

        // let unique = {};
        properties = properties.map((property) => {
            let id = property.GIS_ID;
            property.totalUnits = totalUnits.get(id) ?? 0;
            return property;
        });
        // deduplicate
        // .filter((property) => {
        //     if (property.GIS_ID in unique) {
        //         return false;
        //     }
        //     unique[property.GIS_ID] = true;
        //     return true;
        // });

        radiusScale = d3
            .scaleSqrt()
            .domain([0, d3.max(properties, (d) => d.totalUnits)])
            .range([0, 50]);
    });

    function getCoords(property) {
        let point = new mapboxgl.LngLat(+property.lon, +property.lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    function getTextCoords(property) {
        let point = new mapboxgl.LngLat(+property.lon, +property.lat);
        let { x, y } = map.project(point);
        return { x: x, y: y };
    }

    function handleMouseOver(index) {
        console.log(index);
        hoverSignals = hoverSignals.map((item, i) => {
            if (i === index) {
                return true;
            }
            return false;
        });
    }

    $: hoverSignals = mappedProperties?.map(() => false);

    $: map?.on("move", (evt) => mapViewChanged++);

    let incomeFilter = 100000;
    let yearFilter = 2000;

    $: filteredProperties = properties.filter((property) => {
        return (
            property.YR_BUILT == yearFilter &&
            property.TOTAL_VALUE * 0.32 < incomeFilter
        );
    });

    let mappedProperties = [];

    $: {
        let totalFilteredUnits = d3.rollup(
            filteredProperties,
            (v) => v.length,
            (d) => d.GIS_ID,
        );
        let unique = {};
        mappedProperties = filteredProperties
            .map((property) => {
                let id = property.GIS_ID;
                property.totalFilteredUnits = totalFilteredUnits.get(id) ?? 0;
                return property;
            })
            .filter((property) => {
                if (property.GIS_ID in unique) {
                    return false;
                }
                unique[property.GIS_ID] = true;
                return true;
            });
    }
</script>

<header>
    <div>Test</div>
    <label>
        Enter your annual income:
        <input type="range" min="0" max="1000000" bind:value={incomeFilter} />
        {#if incomeFilter === 1000000}
            <year>$1,000,000+</year>
        {:else}
            <year>${incomeFilter}</year>
        {/if}
    </label>
    <label>
        Year (replace with scrolly):
        <input type="range" min="2000" max="2022" bind:value={yearFilter} />
        <year>{yearFilter}</year>
    </label>
</header>
<div id="mapone" class="map">
    <svg>
        {#key mapViewChanged}
            {#each mappedProperties as property, index}
                <circle
                    {...getCoords(property)}
                    r={radiusScale(property.totalUnits)}
                    fill="green"
                    fill-opacity={property.totalFilteredUnits /
                        property.totalUnits}
                    stroke="white"
                    on:mouseover={() => handleMouseOver(index)}
                    on:mouseleave={() => handleMouseOver(-1)}
                >
                </circle>
            {/each}
        {/key}

        {#key mapViewChanged}
            {#each mappedProperties as property, index}
                {#if hoverSignals[index] === true}
                    <rect
                        {...getTextCoords(property)}
                        width="250"
                        height="120"
                        fill="rgba(0,0,0,0.8)"
                    >
                    </rect>
                    <text {...getTextCoords(property)} fill="#fff">
                        <tspan {...getTextCoords(property)} dx="1em" dy="2em"
                            >{property.ST_NUM}
                            {property.ST_NAME}, {property.CITY}</tspan
                        >
                        <tspan {...getTextCoords(property)} dx="1em" dy="3.75em"
                            >Built: {property.YR_BUILT}</tspan
                        >
                        <tspan {...getTextCoords(property)} dx="1em" dy="5.25em"
                            >Number of Units: {property.totalUnits}</tspan
                        >
                    </text>
                {/if}
            {/each}
        {/key}
    </svg>
</div>

<style>
    @import url("$lib/global.css");

    #mapone {
        height: 500px;
    }
</style>
