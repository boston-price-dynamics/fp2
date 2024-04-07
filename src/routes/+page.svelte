<script>
  import mapboxgl from "mapbox-gl";
  import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import { VERSION } from "svelte/compiler";

  mapboxgl.accessToken =
    "pk.eyJ1Ijoianlvb25zb25nIiwiYSI6ImNsdW95emJzMzIxMDQya3FwcXc1NzA4c2sifQ.C9_2pEU8Z00WnIWNndTg_Q";

  let map;
  let mapViewChanged = 0;

  let properties = [];
  let radiusScale;
  let opacityScale;

  let yearFilter = 1950;

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
    hoverColors = hoverColors.map((color, i) => {
      if (i === index) {
        return "green";
      }
      return "#FFBB05";
    });
  }

  $: map?.on("move", (evt) => mapViewChanged++);

  $: filteredProperties =
    yearFilter === 1950
      ? properties
      : properties.filter((property) => {
          return property.YR_BUILT <= yearFilter;
        });

  $: opacityScale = d3
    .scalePow(3)
    .domain([
      yearFilter === 1950 ? 1850 : yearFilter - 100,
      yearFilter === 1950 ? 2024 : yearFilter,
    ])
    .range([0.05, 0.75]);

  $: hoverSignals = filteredProperties?.map(() => false);
  $: hoverColors = filteredProperties?.map(() => "#FFBB05");

  onMount(async () => {
    map = new mapboxgl.Map({
      container: "map",
      style: "mapbox://styles/jyoonsong/cluq04ktu05cr01qqb4rp4v4f",
      center: [-71.09415, 42.36027],
      zoom: 12,
    });
    await new Promise((resolve) => map.on("load", resolve));

    properties = await d3.csv(
      "https://raw.githubusercontent.com/boston-price-dynamics/fp2/main/assess_lat_long.csv",
    );

    properties.forEach((el) => {
      el.TOTAL_VALUE = +el.TOTAL_VALUE;
    });

    let totalValue = d3.rollup(
      properties,
      (v) => d3.sum(v, (d) => d.TOTAL_VALUE),
      (d) => d.GIS_ID,
    );

    let totalUnits = d3.rollup(
      properties,
      (v) => v.length,
      (d) => d.GIS_ID,
    );

    let unique = {};
    properties = properties
      .map((property) => {
        let id = property.GIS_ID;
        property.totalValue = totalValue.get(id) ?? 0;
        property.totalUnits = totalUnits.get(id) ?? 0;
        return property;
      })
      // deduplicate
      .filter((property) => {
        if (property.GIS_ID in unique) {
          return false;
        }
        unique[property.GIS_ID] = true;
        return true;
      });

    properties = properties.filter((property) => {
      return (
        property.totalValue > 100_000_000 &&
        property.totalValue / property.totalUnits > 1_000_000
      );
    });
    console.log(properties);

    radiusScale = d3
      .scaleSqrt()
      .domain([0, d3.max(properties, (d) => d.totalValue)])
      .range([0, 40]);

    // let totalUnits = d3.rollup(properties);
  });
</script>

<nav>
  <h3>🏠</h3>
</nav>
<header>
  <div>
    <h4>
      Boston Luxury Developments (Total Assessed Value > $10m, Average Assessed
      Value > $1m)
    </h4>
    <p>Assessed Values</p>
  </div>
  <label>
    Year:
    <input type="range" min="1950" max="2024" bind:value={yearFilter} />
    {#if yearFilter === 1950}
      <em>(any year)</em>
    {:else}
      <year>{yearFilter}</year>
    {/if}
  </label>
</header>
<div id="map">
  <svg>
    {#key mapViewChanged}
      {#each filteredProperties as property, index}
        <circle
          {...getCoords(property)}
          r={radiusScale(property.totalValue)}
          fill={hoverColors[index]}
          fill-opacity={hoverSignals[index]
            ? 0.8
            : opacityScale(property.YR_BUILT)}
          stroke="white"
          on:mouseover={() => handleMouseOver(index)}
          on:mouseleave={() => handleMouseOver(-1)}
        >
        </circle>
      {/each}
    {/key}

    {#key mapViewChanged}
      {#each filteredProperties as property, index}
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
              >{property.ST_NUM} {property.ST_NAME}, {property.CITY}</tspan
            >
            <tspan {...getTextCoords(property)} dx="1em" dy="3.75em"
              >Built: {property.YR_BUILT}</tspan
            >
            <tspan {...getTextCoords(property)} dx="1em" dy="5.25em"
              >Number of Units: {property.totalUnits}</tspan
            >
            <tspan {...getTextCoords(property)} dx="1em" dy="6.75em"
              >Average Assessed Value: ${(
                property.totalValue / property.totalUnits
              ).toLocaleString(undefined, { maximumFractionDigits: 0 })}</tspan
            >
            <tspan {...getTextCoords(property)} dx="1em" dy="8.25em"
              >Total Assessed Value: ${property.totalValue.toLocaleString(
                undefined,
                { maximumFractionDigits: 0 },
              )}</tspan
            >
          </text>
        {/if}
      {/each}
    {/key}
  </svg>
</div>

<style>
  @import url("$lib/global.css");
</style>
