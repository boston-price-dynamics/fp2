<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from "d3";
    import Scrolly from "svelte-scrolly";
    import Countup from "svelte-countup";

    mapboxgl.accessToken =
        "pk.eyJ1Ijoianlvb25zb25nIiwiYSI6ImNsdW95emJzMzIxMDQya3FwcXc1NzA4c2sifQ.C9_2pEU8Z00WnIWNndTg_Q";

    let map;
    let mapViewChanged = 0;

    let properties = [];
    let incomes = [];
    let incomesToCdf = {};
    let houseprices = [];
    let housepricesToCdf = {};
    let radiusScale;
    onMount(async () => {
        map = new mapboxgl.Map({
            container: "mapone",
            style: "mapbox://styles/jyoonsong/cluq04ktu05cr01qqb4rp4v4f",
            center: [-71.1007596, 42.3249561],
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

        incomes = await d3.csv(
            "https://raw.githubusercontent.com/boston-price-dynamics/fp2/main/income_dist.csv",
            d3.autoType,
        );

        for (let it of incomes) {
            incomesToCdf[it.income] = it.cdf;
        }

        houseprices = await d3.csv(
            "https://raw.githubusercontent.com/boston-price-dynamics/fp2/main/house_dist.csv",
            d3.autoType,
        );

        for (let ht of houseprices) {
            housepricesToCdf[ht.house_price] = ht.cdf;
        }
        console.log(housepricesToCdf);
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
        var x = d3.scaleLinear().domain([0, 5_000_000]).range([0, 800]);
        return x(value);
    }

    function handleCy(value, index) {
        var max = 1800;
        var y = d3.scaleLinear().domain([0, max]).range([270, 0]);

        const range = filteredPropertiesByYear.filter(
            (p) =>
                p.TOTAL_VALUE <= parseInt(value) + 100_000 &&
                p.TOTAL_VALUE >= parseInt(value) - 100_000,
        );
        const multiplier = index % 2 === 0 ? 1 : -1;
        const result = max / 2 + multiplier * Math.random() * range.length;
        return y(result);
    }

    function handleColor(value, incomeFilter) {
        if (value * 0.3 < incomeFilter) {
            return "#1a9850";
        }
        return "#d73027";
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

    $: filteredPropertiesByYear = properties.filter((property) => {
        return property.YR_BUILT == yearFilter;
    });

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
        let totalValueYear = d3.rollup(
            filteredPropertiesYear,
            (v) => d3.sum(v, (d) => d.TOTAL_VALUE),
            (d) => d.GIS_ID,
        );
        filteredPropertiesYear = filteredPropertiesYear.map((property) => {
            let id = property.GIS_ID;
            property.totalUnitsYear = totalUnitsYear.get(id) ?? 0;
            property.totalValueYear = totalValueYear.get(id) ?? 0;
            return property;
        });

        filteredPropertiesYearIncome = filteredPropertiesYear.filter(
            (property) => {
                return property.TOTAL_VALUE * 0.3 < incomeFilter;
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
                        return property.TOTAL_VALUE * 0.3 < incomeFilter;
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

    /////////////////

    let width = 500;
    let height = 200;
    let padding = { top: 20, right: 15, bottom: 20, left: 25 };
    let xTicks = [0, 200000, 400000, 600000, 800000, 1000000];

    $: xScale = d3
        .scaleLinear()
        .domain([minX, maxX])
        .range([padding.left, width - padding.right]);

    $: yScale = d3
        .scaleLinear()
        .domain(d3.extent(incomes, (d) => d.pdf))
        .range([height - padding.bottom, padding.top]);

    $: minX = incomes[0]?.income;
    $: maxX = incomes[incomes.length - 1]?.income;
    $: path = `M${incomes.map((p) => `${xScale(p.income)},${yScale(p.pdf)}`).join("L")}`;
    $: area = `${path}L${xScale(maxX)},${yScale(0)}L${xScale(minX)},${yScale(0)}Z`;

    /////////
    let xTicks2 = [0, 1000000, 2000000, 3000000, 4000000, 5000000];
    $: xScale2 = d3
        .scaleLinear()
        .domain([minX2, maxX2])
        .range([padding.left, width - padding.right]);

    $: yScale2 = d3
        .scaleLinear()
        .domain(d3.extent(houseprices, (d) => d.pdf))
        .range([height - padding.bottom, padding.top]);

    $: minX2 = houseprices[0]?.house_price;
    $: maxX2 = houseprices[houseprices.length - 1]?.house_price;
    $: path2 = `M${houseprices.map((p) => `${xScale2(p.house_price)},${yScale2(p.pdf)}`).join("L")}`;
    $: area2 = `${path2}L${xScale2(maxX)},${yScale2(0)}L${xScale2(minX)},${yScale2(0)}Z`;
</script>

<Scrolly
    bind:progress={yearProgress}
    --scrolly-layout="viz-first"
    threshold="0"
>
    <svelte:fragment slot="viz">
        <header>
            <div>
                <h4>New Developments in Boston</h4>
                <p>One Dot per Development</p>
            </div>
            <label>
                Enter your annual household income:
                <input
                    type="range"
                    min="0"
                    max="1000000"
                    bind:value={incomeFilter}
                />
                <year>{USDollar.format(incomeFilter)}</year>
            </label>
        </header>
        <div class="summary">
            <h3>
                With your income of <span class="value">
                    <year>{USDollar.format(incomeFilter)}</year>
                </span>
                you can afford
                <span class="value"
                    >{(
                        (statsByYear[yearFilter]?.number_affordable /
                            statsByYear[yearFilter]?.number_built) *
                        100
                    ).toLocaleString(undefined, {
                        maximumFractionDigits: 2,
                    })}%</span
                >
                of the units built in <span class="value">{yearFilter}</span>
            </h3>
        </div>
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
                                    >New units built: {property.totalUnitsYear}</tspan
                                >
                                <tspan
                                    {...getTextCoords(property)}
                                    dx="1em"
                                    dy="5.25em"
                                    >New units you can afford: {property.totalUnitsYearIncome}</tspan
                                >
                                <tspan
                                    {...getTextCoords(property)}
                                    dx="1em"
                                    dy="6.75em"
                                    >% units you can afford: {(
                                        (property.totalUnitsYearIncome /
                                            property.totalUnitsYear) *
                                        100
                                    ).toLocaleString(undefined, {
                                        maximumFractionDigits: 2,
                                    })}%</tspan
                                >
                                <tspan
                                    {...getTextCoords(property)}
                                    dx="1em"
                                    dy="8.25em"
                                    >Average new unit price:
                                    {USDollar.format(
                                        property.totalValueYear /
                                            property.totalUnitsYear,
                                    )}
                                </tspan>
                            </text>
                        {/if}
                    {/each}
                {/key}

                {#each [10, 50, 120] as sz, index}
                    <circle
                        cx="50"
                        cy={radiusScale ? 80 - radiusScale(sz) : 0}
                        r={radiusScale ? radiusScale(sz) : 0}
                        stroke="black"
                        fill="none"
                    ></circle>
                    <line
                        x1={radiusScale ? 50 + radiusScale(sz) : 0}
                        x2="80"
                        y1={radiusScale ? 80 - radiusScale(sz) : 0}
                        y2={radiusScale ? 80 - radiusScale(sz) : 0}
                        stroke="black"
                    ></line>
                    <text
                        x="80"
                        y={radiusScale ? 80 - radiusScale(sz) : 0}
                        font-size="10"
                        alignment-baseline="middle">{sz}</text
                    >
                {/each}
            </svg>
        </div>
        <div id="map_legend">
            <div>% Affordable:</div>
            {#each [0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1] as color, index}
                <div style="--color: {colorScale(color)}">{color * 100}%</div>
            {/each}
        </div>
        <div id="scatter">
            <svg width="800" height="270">
                <g transform="translate(10,0)">
                    <!-- X axis -->
                    <g
                        transform="translate(0,210)"
                        fill="none"
                        font-size="10"
                        font-family="sans-serif"
                        text-anchor="middle"
                        ><path
                            class="domain"
                            stroke="currentColor"
                            d="M0,6V0H760V6"
                        ></path><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">0M</text
                            ></g
                        ><g class="tick" opacity="1" transform="translate(76,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">0.5M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(152,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">1M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(228,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">1.5M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(304,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(380,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2.5M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(456,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">3M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(532,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">3.5M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(608,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">4M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(684,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">4.5M</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(760,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">5M</text
                            ></g
                        >
                        <text
                            transform="translate(380,37)"
                            fill="#000"
                            style="font-size: 12px;"
                            >House Price (One Dot per Unit)</text
                        >
                    </g>
                    <!-- SCATTER -->
                    <g transform="translate(0, -30)">
                        {#each filteredPropertiesByYear as property, index}
                            <circle
                                cx={handleCx(property.TOTAL_VALUE)}
                                cy={handleCy(property.TOTAL_VALUE, index)}
                                r="3"
                                opacity="0.5"
                                fill={handleColor(
                                    property.TOTAL_VALUE,
                                    incomeFilter,
                                )}
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
                <div class="info">
                    <h1>{year}</h1>
                    <div class="stat">
                        <p>New units built:</p>
                        <h2>
                            {(statsByYear[year]?.number_built).toLocaleString()}
                        </h2>
                    </div>
                    <div class="stat">
                        <p>New units you can afford:</p>
                        <h2>
                            {(statsByYear[
                                year
                            ]?.number_affordable).toLocaleString()}
                        </h2>
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
                        <h2>
                            {USDollar.format(statsByYear[year]?.median_value)}
                        </h2>
                    </div>
                </div>
            </section>
        {/each}
    </div>
</Scrolly>
<div class="comb_prob">
    <div class="prob_section">
        <div class="chart">
            <svg>
                <path class="path-area" d={area} />
                <path class="path-line" d={path} />
                <g class="axis x-axis">
                    {#each xTicks as tick}
                        <g
                            class="tick tick-{tick}"
                            transform="translate({xScale(tick)},{height})"
                        >
                            <line
                                y1="-{height}"
                                y2="-{padding.bottom}"
                                x1="0"
                                x2="0"
                            />
                            <text y="-2" font-size="10"
                                >{width > 380
                                    ? USDollar.format(tick)
                                    : formatMobile(tick)}</text
                            >
                        </g>
                    {/each}
                    <g
                        class="tick highlight-tick"
                        transform="translate({xScale(incomeFilter)},{height})"
                    >
                        <line
                            y1="-{height}"
                            y2="-{padding.bottom}"
                            x1="0"
                            x2="0"
                        />
                    </g>
                </g>
            </svg>
        </div>
        <h3>
            Your household income is greater than <b
                >{Math.min(
                    incomesToCdf[Math.round(incomeFilter / 5000) * 5000] * 100,
                    99,
                )?.toLocaleString(undefined, {
                    maximumFractionDigits: 2,
                }) ?? 0}%</b
            > of other Boston households...
        </h3>
    </div>
    <div class="prob_section">
        <div class="chart">
            <svg>
                <path class="path-area" d={area2} />
                <path class="path-line" d={path2} />
                <g class="axis x-axis">
                    {#each xTicks2 as tick}
                        <g
                            class="tick tick-{tick}"
                            transform="translate({xScale2(tick)},{height})"
                        >
                            <line
                                y1="-{height}"
                                y2="-{padding.bottom}"
                                x1="0"
                                x2="0"
                            />
                            <text y="-2" font-size="10"
                                >{width > 380
                                    ? USDollar.format(tick)
                                    : formatMobile(tick)}</text
                            >
                        </g>
                    {/each}
                    <g
                        class="tick highlight-tick"
                        transform="translate({xScale2(
                            incomeFilter / 0.3,
                        )},{height})"
                    >
                        <line
                            y1="-{height}"
                            y2="-{padding.bottom}"
                            x1="0"
                            x2="0"
                        />
                    </g>
                </g>
            </svg>
        </div>
        <h3>
            ...yet you can only afford <b
                >{Math.min(
                    housepricesToCdf[
                        Math.round(incomeFilter / 0.3 / 10000) * 10000
                    ] * 100,
                    99,
                )?.toLocaleString(undefined, {
                    maximumFractionDigits: 2,
                }) ?? 0}%</b
            >
            of new housing units developed between 2000 and 2022.
        </h3>
    </div>
</div>

<style>
    @import url("$lib/global.css");

    #mapone {
        height: 400px;
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

    .chart {
        width: 100%;
        max-width: 500px;
    }

    .chart svg {
        position: relative;
        width: 100%;
        height: 200px;
        overflow: visible;
    }
    .tick {
        font-size: 0.725em;
        font-weight: 200;
    }

    /* .tick line {
        stroke: #888;
        stroke-dasharray: 2;
    } */

    .tick text {
        fill: #888;
        text-anchor: start;
    }

    .tick.tick-0 line {
        stroke: #888;
        stroke-dasharray: 0;
    }

    .tick.highlight-tick line {
        stroke: #ffee91;
        stroke-dasharray: 0;
        stroke-width: 3;
    }

    .x-axis .tick text {
        text-anchor: middle;
    }
    .path-line {
        fill: none;
        stroke: rgb(55, 183, 225);
        stroke-linejoin: round;
        stroke-linecap: round;
        stroke-width: 2;
    }
    .path-area {
        fill: rgb(55, 183, 225, 0.2);
    }
    .prob_section {
        display: flex;
        flex-direction: column;
        align-items: center;
    }
    .comb_prob {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 3em;
        background-color: #2c2f39;
        color: white;
        padding: 4rem 1rem;
        margin: 2rem 5rem;
    }
    h3 b {
        font-size: 40px;
        padding: 0 6px;
    }
</style>
