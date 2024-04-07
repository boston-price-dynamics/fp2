<script>
  import mapboxgl from "mapbox-gl";
  import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  import { onMount } from "svelte";
  import * as d3 from "d3";

  mapboxgl.accessToken =
    "pk.eyJ1Ijoianlvb25zb25nIiwiYSI6ImNsdW95emJzMzIxMDQya3FwcXc1NzA4c2sifQ.C9_2pEU8Z00WnIWNndTg_Q";

  let map;
  let stations = [];
  let trips = [];
  let mapViewChanged = 0;
  let departures;
  let arrivals;
  let radiusScale = 0;
  let yearFilter = 2017;

  function getCoords(station) {
    let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
    let { x, y } = map.project(point);
    return { cx: x, cy: y };
  }

  $: radiusScale = d3
    .scaleSqrt()
    .domain([0, d3.max(stations, (d) => d.totalTraffic)])
    .range([0, 25]);

  // listener
  $: map?.on("move", (evt) => mapViewChanged++);

  onMount(async () => {
    map = new mapboxgl.Map({
      /* options */
      container: "map",
      style: "mapbox://styles/mapbox/streets-v12",
      zoom: 12,
      center: [-71.09415, 42.36027],
    });

    await new Promise((resolve) => map.on("load", resolve));

    // blue bike stations
    stations = await d3.csv(
      "https://vis-society.github.io/labs/8/data/bluebikes-stations.csv"
    );

    // blue bike station traffic
    trips = await d3.csv(
      "https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv"
    );

    departures = d3.rollup(
      trips,
      (v) => v.length,
      (d) => d.start_station_id
    );
    arrivals = d3.rollup(
      trips,
      (v) => v.length,
      (d) => d.end_station_id
    );
    stations = stations.map((station) => {
      let id = station.Number;
      station.departures = departures.get(id) ?? 0;
      station.arrivals = arrivals.get(id) ?? 0;
      station.totalTraffic = departures.get(id) + arrivals.get(id) ?? 0;
      return station;
    });
  });
</script>

<h1>🏠 Boston Price Dynamics</h1>
<header>
  <label>
    Filter by year:
    <input type="range" min="1989" max="2024" bind:value={yearFilter} />
    {#if yearFilter === 1989}
      <em>(any year)</em>
    {:else}
      <year>{yearFilter}</year>
    {/if}
  </label>
</header>
<div id="map">
  <svg>
    {#key mapViewChanged}
      {#each stations as station}
        <circle
          {...getCoords(station)}
          r={radiusScale(station.totalTraffic)}
          fill="steelblue"
          fill-opacity="0.6"
          stroke="white"
        >
          <title
            >{station.totalTraffic} trips ({station.departures} departures, {station.arrivals}
            arrivals)</title
          >
        </circle>
      {/each}
    {/key}
  </svg>
</div>

<style>
  @import url("$lib/global.css");
</style>
