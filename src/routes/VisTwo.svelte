<script>
    import * as d3 from "d3";
    import { onMount } from "svelte";
    import Scrolly from "svelte-scrolly";
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";

    // mapboxgl.accessToken =
    //     "pk.eyJ1IjoiY3JvemFyaW8iLCJhIjoiY2x1c3E3c3RpMDBnejJrb2c1NDh5a3VwcSJ9.UGIxS93W_cbBMHzrWcIV6A";

    let timelineData = [];
    let map;
    let buildings = [];
    let filteredBuildings;

    function getCoords(building) {
        let lat = building.lat;
        let long = building.long;
        let point = new mapboxgl.LngLat(long, lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }
    let mapViewChanged = 0;
    $: map?.on("move", (evt) => mapViewChanged++);

    onMount(async () => {
        map = new mapboxgl.Map({
            container: "map",
            style: "mapbox://styles/mapbox/standard",
            center: [-71.08401, 42.34577],
            zoom: 16,
            pitch: 60,
        });
        await new Promise((resolve) => map.on("load", resolve));

        timelineData = await d3.json("One Dalton.json");
        timelineData.forEach((row) => {
            row.dt = new Date(row.date);
        });

        buildings = await d3.csv("/full_years.csv");
    });

    let timelineProgress = 100;
    let mapMaxtime = 2013;
    $: timeScale = d3
        .scaleTime()
        .domain(d3.extent(timelineData, (evt) => evt.dt))
        .range([0, 100]);
    $: mapMaxtime = timeScale.invert(timelineProgress).getFullYear();

    $: {
        filteredBuildings = buildings.filter((building) => {
            return new Date(building.Year).getFullYear() === mapMaxtime;
        });
    }
    let colorScale = d3
        .scaleLinear()
        .domain([0, 3000000])
        .range(["white", "black"]); // Adjust the range of colors as needed
</script>

<Scrolly bind:progress={timelineProgress}>
    {#each timelineData as item, index}
        <div class="descrips" style="text-align: center;">
            <!-- <h1 class="date-heading">{item.date.toISOString().split('T')[0]}</h1> -->
            <h1 class="date-heading">{item.date}</h1>
            <h2>{item.event}</h2>
            <h3>{item.description}</h3>
            {#if item.image !== ""}
                <img src={item.image} alt="img" />
            {/if}
            <h5><a href={item.link}>{item.link}</a></h5>
            <br />
            <br />
            <br />
            <br />
            <br />
            <br />
            <br />
            <br />
            <br />
            <br />
            <br />
            <br />
        </div>
    {/each}

    <svelte:fragment slot="viz">
        <p>{mapMaxtime}</p>
        <div id="map">
            <svg>
                {#key mapViewChanged}
                    {#each filteredBuildings as building}
                        <circle
                            {...getCoords(building)}
                            r="7"
                            fill={colorScale(
                                parseInt(building.TOTAL_VALUE.replace(/,/g, ""))
                            )}
                        />
                    {/each}
                {/key}
            </svg>
        </div>
    </svelte:fragment>
</Scrolly>

<style>
    @import url("$lib/global.css");
    #map {
        flex: 1;
        min-height: 70vh;
        min-width: 50%;
    }
    .descrips {
        min-height: 80vh;
        max-height: 80vh;
        /* max-width: 50%; */
    }

    .date-heading {
        font-weight: bold;
        font-size: 30px; /* You can adjust the font size as needed */
    }

    img {
        max-width: 50%;
        min-width: 40%;
    }

    #map circle {
        pointer-events: auto;
    }
</style>
