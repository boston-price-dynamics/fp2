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
            "https://raw.githubusercontent.com/boston-price-dynamics/fp2/main/assess_lat_long.csv"
        );

        let totalUnits = d3.rollup(
            properties,
            (v) => v.length,
            (d) => d.GIS_ID
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

        // set the dimensions and margins of the graph
        var margin = { top: 10, right: 30, bottom: 30, left: 60 },
            width = 460 - margin.left - margin.right,
            height = 400 - margin.top - margin.bottom;

        // append the svg object to the body of the page
        var svg = d3
            .select("#scatter")
            .append("svg")
            .attr("width", width + margin.left + margin.right)
            .attr("height", height + margin.top + margin.bottom)
            .append("g")
            .attr(
                "transform",
                "translate(" + margin.left + "," + margin.top + ")"
            );

        // Add X axis
        var x = d3.scaleLinear().domain([0, 5_000_000]).range([0, width]);
        svg.append("g")
            .attr("transform", "translate(0," + height + ")")
            .call(
                d3
                    .axisBottom(x)
                    .tickFormat((d, i) => (d === 0 ? 0 : d / 1_000_000 + "M"))
            );

        // Add Y axis
        var y = d3
            .scaleLinear()
            .domain([0, propertiesByYear.length])
            .range([height, 0]);
        // svg.append("g").call(d3.axisLeft(y));

        const half = propertiesByYear.length / 2;

        // Add dots
        svg.append("g")
            .selectAll("dot")
            .data(propertiesByYear)
            .enter()
            .append("circle")
            .attr("cx", function (d) {
                return x(d.TOTAL_VALUE);
            })
            .attr("cy", function (d, i) {
                let range = propertiesByYear.filter(
                    (p) =>
                        p.TOTAL_VALUE <= parseInt(d.TOTAL_VALUE) + 100_000 &&
                        p.TOTAL_VALUE >= parseInt(d.TOTAL_VALUE) - 100_000
                );
                const result =
                    i % 2 === 0
                        ? half + Math.random() * range.length
                        : half - Math.random() * range.length;
                return y(result);
                // return y(10);
            })
            .attr("r", 1.5)
            .style("fill", "#69b3a2");

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

    $: propertiesByYear = properties.filter(
        (p) => parseInt(p.YR_BUILT) === yearFilter
    );

    let mappedProperties = [];

    $: {
        let totalFilteredUnits = d3.rollup(
            filteredProperties,
            (v) => v.length,
            (d) => d.GIS_ID
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

<div id="scatter">
    <svg width="460" height="400">
        <g transform="translate(60, 10)">
            <!-- X AXIS -->
            <g
                transform="translate(0,360)"
                fill="none"
                font-size="10"
                font-family="sans-serif"
                text-anchor="middle"
            >
                <path class="domain" stroke="currentColor" d="M0,6V0H370V6"
                ></path>
                <g class="tick" opacity="1" transform="translate(0,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">0</text
                    ></g
                ><g class="tick" opacity="1" transform="translate(37,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">0.5M</text
                    ></g
                ><g class="tick" opacity="1" transform="translate(74,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">1M</text
                    ></g
                ><g class="tick" opacity="1" transform="translate(111,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">1.5M</text
                    ></g
                ><g class="tick" opacity="1" transform="translate(148,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">2M</text
                    ></g
                ><g class="tick" opacity="1" transform="translate(185,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">2.5M</text
                    ></g
                ><g class="tick" opacity="1" transform="translate(222,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">3M</text
                    ></g
                ><g class="tick" opacity="1" transform="translate(259,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">3.5M</text
                    ></g
                ><g class="tick" opacity="1" transform="translate(296,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">4M</text
                    ></g
                ><g class="tick" opacity="1" transform="translate(333,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">4.5M</text
                    ></g
                ><g class="tick" opacity="1" transform="translate(370,0)"
                    ><line stroke="currentColor" y2="6"></line><text
                        fill="currentColor"
                        y="9"
                        dy="0.71em">5M</text
                    ></g
                ></g
            >
            <!-- SCATTER -->
            <g> </g>
        </g>
    </svg>
</div>

<style>
    @import url("$lib/global.css");

    #mapone {
        height: 500px;
    }
</style>
