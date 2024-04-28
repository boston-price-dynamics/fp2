<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from "d3";
    import Scrolly from "svelte-scrolly";

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
            center: [-71.1007596, 42.3149367],
            zoom: 11,
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

        properties = properties.map((property) => {
            let id = property.GIS_ID;
            property.totalUnits = totalUnits.get(id) ?? 0;
            return property;
        });

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
        hoverSignals = hoverSignals.map((item, i) => {
            if (i === index) {
                return true;
            }
            return false;
        });
    }

    function handleCx(value) {
        var x = d3.scaleLinear().domain([0, 5_000_000]).range([0, 460]);
        return x(value);
    }

    function handleCy(value, index) {
        var y = d3.scaleLinear().domain([0, 2000]).range([400, 0]);

        const range = filteredPropertiesYear.filter(
            (p) =>
                p.TOTAL_VALUE <= parseInt(value) + 100_000 &&
                p.TOTAL_VALUE >= parseInt(value) - 100_000,
        );
        const multiplier = index % 2 === 0 ? 1 : -1;
        const result = 1000 + multiplier * Math.random() * range.length;
        return y(result);
    }

    $: map?.on("move", (evt) => mapViewChanged++);

    let incomeFilter = 100000;

    let progressToYearScale = d3
        .scaleLinear()
        .domain([0, 100])
        .rangeRound([2000, 2022]);
    $: yearFilter = progressToYearScale(yearProgress);

    let years = [];
    for (let year = 2000; year <= 2022; year++) {
        years.push(year);
    }

    let filteredPropertiesYear;
    let filteredPropertiesYearIncome;
    let hoverSignals;
    let yearProgress;
    $: {
        filteredPropertiesYear = properties.filter((property) => {
            return property.YR_BUILT == yearFilter;
        });
        let totalUnitsYear = d3.rollup(
            filteredPropertiesYear,
            (v) => v.length,
            (d) => d.GIS_ID,
        );
        filteredPropertiesYear = filteredPropertiesYear.map((property) => {
            let id = property.GIS_ID;
            property.totalUnitsYear = totalUnitsYear.get(id) ?? 0;
            return property;
        });

        filteredPropertiesYearIncome = filteredPropertiesYear.filter(
            (property) => {
                return incomeFilter === 1000000
                    ? true
                    : property.TOTAL_VALUE * 0.32 < incomeFilter;
            },
        );
        let totalUnitsYearIncome = d3.rollup(
            filteredPropertiesYearIncome,
            (v) => v.length,
            (d) => d.GIS_ID,
        );
        let unique = {};
        filteredPropertiesYear = filteredPropertiesYear
            .map((property) => {
                let id = property.GIS_ID;
                property.totalUnitsYearIncome =
                    totalUnitsYearIncome.get(id) ?? 0;
                return property;
            })
            .filter((property) => {
                if (property.GIS_ID in unique) {
                    return false;
                }
                unique[property.GIS_ID] = true;
                return true;
            });

        hoverSignals = filteredPropertiesYear?.map(() => false);
    }

    let colorScale = d3
        .scaleLinear()
        .domain([0, 0.5, 1])
        .range(["#d73027", "#ffffbf", "#1a9850"]);

    let statsByYear = {};
    $: {
        for (let year of years) {
            if (Math.abs(year - yearFilter) < 2) {
                let _filteredPropertiesYear = properties.filter((property) => {
                    return property.YR_BUILT == year;
                });
                let _filteredPropertiesYearIncome =
                    _filteredPropertiesYear.filter((property) => {
                        return incomeFilter === 1000000
                            ? true
                            : property.TOTAL_VALUE * 0.32 < incomeFilter;
                    });
                statsByYear[year] = {
                    number_built: _filteredPropertiesYear.length,
                    number_affordable: _filteredPropertiesYearIncome.length,
                    median_value: d3.median(
                        _filteredPropertiesYear,
                        (d) => d.TOTAL_VALUE,
                    ),
                };
            }
        }
    }
    let USDollar = new Intl.NumberFormat("en-US", {
        style: "currency",
        currency: "USD",
        maximumFractionDigits: 0,
    });
</script>

<Scrolly
    bind:progress={yearProgress}
    --scrolly-layout="viz-first"
    threshold="0"
>
    <svelte:fragment slot="viz">
        <header>
            <div>
                <h4>New Developments in Boston in {yearFilter}</h4>
                <p>Source: Boston Property Assessment Data</p>
            </div>
            <label>
                Enter your annual income:
                <input
                    type="range"
                    min="0"
                    max="1000000"
                    bind:value={incomeFilter}
                />
                {#if incomeFilter === 1000000}
                    <year>$1,000,000+</year>
                {:else}
                    <year>{USDollar.format(incomeFilter)}</year>
                {/if}
            </label>
        </header>
        <div id="mapone" class="map">
            <svg>
                {#key mapViewChanged}
                    {#each filteredPropertiesYear as property, index}
                        <circle
                            {...getCoords(property)}
                            r={radiusScale(property.totalUnitsYear)}
                            fill={colorScale(
                                property.totalUnitsYearIncome /
                                    property.totalUnitsYear,
                            )}
                            fill-opacity="0.7"
                            stroke="white"
                            on:mouseover={() => handleMouseOver(index)}
                            on:mouseleave={() => handleMouseOver(-1)}
                        >
                        </circle>
                    {/each}
                {/key}

                {#key mapViewChanged}
                    {#each filteredPropertiesYear as property, index}
                        {#if hoverSignals[index] === true}
                            <rect
                                {...getTextCoords(property)}
                                width="250"
                                height="120"
                                fill="rgba(0,0,0,0.8)"
                            >
                            </rect>
                            <text {...getTextCoords(property)} fill="#fff">
                                <tspan
                                    {...getTextCoords(property)}
                                    dx="1em"
                                    dy="2em"
                                    >{property.ST_NUM}
                                    {property.ST_NAME}, {property.CITY}</tspan
                                >
                                <tspan
                                    {...getTextCoords(property)}
                                    dx="1em"
                                    dy="3.75em"
                                    >Built: {property.YR_BUILT}</tspan
                                >
                                <tspan
                                    {...getTextCoords(property)}
                                    dx="1em"
                                    dy="5.25em"
                                    ># Units: {property.totalUnitsYear}</tspan
                                >
                                <tspan
                                    {...getTextCoords(property)}
                                    dx="1em"
                                    dy="6.75em"
                                    ># Units Afforded: {property.totalUnitsYearIncome}</tspan
                                >
                            </text>
                        {/if}
                    {/each}
                {/key}
            </svg>
        </div>
        <div id="map_legend">
            <div>% Affordable:</div>
            {#each [0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1] as color, index}
                <div style="--color: {colorScale(color)}">{color * 100}%</div>
            {/each}
        </div>
        <div id="size_legend">
            <svg width="100" height="100">
                {#each [10, 50, 120] as sz, index}
                    <circle
                        cx="50"
                        cy={radiusScale ? 90 - radiusScale(sz) : 0}
                        r={radiusScale ? radiusScale(sz) : 0}
                        stroke="black"
                        fill="none"
                    ></circle>
                    <line
                        x1={radiusScale ? 50 + radiusScale(sz) : 0}
                        x2="80"
                        y1={radiusScale ? 90 - radiusScale(sz) : 0}
                        y2={radiusScale ? 90 - radiusScale(sz) : 0}
                        stroke="black"
                    ></line>
                    <text
                        x="80"
                        y={radiusScale ? 90 - radiusScale(sz) : 0}
                        font-size="10"
                        alignment-baseline="middle">{sz}</text
                    >
                {/each}
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
                        <path
                            class="domain"
                            stroke="currentColor"
                            d="M0,6V0H370V6"
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
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(111,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">1.5M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(148,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(185,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2.5M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(222,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">3M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(259,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">3.5M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(296,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">4M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(333,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">4.5M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(370,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">5M</text
                            ></g
                        >
                    </g>
                    <!-- SCATTER -->
                    <g>
                        {#each filteredPropertiesYear as property, index}
                            <circle
                                cx={handleCx(property.TOTAL_VALUE)}
                                cy={handleCy(property.TOTAL_VALUE, index)}
                                r="1.5"
                                fill="#69b3a2"
                            ></circle>
                        {/each}
                    </g>
                </g>
            </svg>
        </div>
    </svelte:fragment>
    <div>
        {#each years as year, index}
            <section>
                <h1>{year}</h1>
                <div class="stat">
                    <p>New units built:</p>
                    <h2>{statsByYear[year]?.number_built}</h2>
                </div>
                <div class="stat">
                    <p>New units you can afford:</p>
                    <h2>{statsByYear[year]?.number_affordable}</h2>
                </div>
                <div class="stat">
                    <p>% units you can afford:</p>
                    <h2>
                        {(
                            (statsByYear[year]?.number_affordable /
                                statsByYear[year]?.number_built) *
                            100
                        ).toLocaleString(undefined, {
                            maximumFractionDigits: 2,
                        })}%
                    </h2>
                </div>
                <div class="stat">
                    <p>Median new unit price:</p>
                    <h2>{USDollar.format(statsByYear[year]?.median_value)}</h2>
                </div>
            </section>
        {/each}
    </div>
</Scrolly>

<style>
    @import url("$lib/global.css");

    #mapone {
        height: 500px;
        width: 800px;
    }

    section {
        padding-top: 50px;
        padding-bottom: 300px;
    }

    section h1 {
        font-size: 100px;
        margin: 0;
    }

    #map_legend {
        display: flex;
        justify-content: space-between;
        gap: 1px;
        margin-top: 5px;

        & div {
            flex: 1;
            text-align: center;
            background-color: var(--color);
            font-size: 9px;
        }
    }
</style>
