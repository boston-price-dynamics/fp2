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
            container: "viz2",
            style: "mapbox://styles/jyoonsong/cluq04ktu05cr01qqb4rp4v4f",
            center: [-71.08059, 42.3453683],
            zoom: 16,
            pitch: 60,
        });

        map.on("style.load", () => {
            // Insert the layer beneath any symbol layer.
            const layers = map.getStyle().layers;
            const labelLayerId = layers.find(
                (layer) =>
                    layer.type === "symbol" && layer.layout["text-field"],
            ).id;

            // The 'building' layer in the Mapbox Streets
            // vector tileset contains building height data
            // from OpenStreetMap.
            map.addLayer(
                {
                    id: "add-3d-buildings",
                    source: "composite",
                    "source-layer": "building",
                    filter: ["==", "extrude", "true"],
                    type: "fill-extrusion",
                    minzoom: 15,
                    paint: {
                        "fill-extrusion-color": [
                            "case",
                            ["boolean", ["feature-state", "hover"], false],
                            "#000",
                            "#ddd",
                        ],

                        // Use an 'interpolate' expression to
                        // add a smooth transition effect to
                        // the buildings as the user zooms in.
                        "fill-extrusion-height": [
                            "interpolate",
                            ["linear"],
                            ["zoom"],
                            15,
                            0,
                            15.05,
                            ["get", "height"],
                        ],
                        "fill-extrusion-base": [
                            "interpolate",
                            ["linear"],
                            ["zoom"],
                            15,
                            0,
                            15.05,
                            ["get", "min_height"],
                        ],
                        "fill-extrusion-opacity": 0.8,
                    },
                },
                labelLayerId,
            );
        });
        await new Promise((resolve) => map.on("load", resolve));

        map.setFeatureState(
            {
                source: "composite",
                sourceLayer: "building",
                id: 818117637,
            },
            {
                hover: true,
            },
        );

        // let fHover;

        // map.on("click", function (e) {
        //     var features = map.queryRenderedFeatures(e.point, {
        //         layers: ["add-3d-buildings"],
        //     });
        //     console.log(features[0].id);
        // });

        // map.on("mousemove", function (e) {
        //     var features = map.queryRenderedFeatures(e.point, {
        //         layers: ["add-3d-buildings"],
        //     });
        //     //we will change pointer and color for 42455719
        //     if (features[0] && features[0].id == 818117637) {
        //         mouseover(features[0]);
        //     } else {
        //         mouseout();
        //     }
        // });

        // map.on("mouseout", function (e) {
        //     mouseout();
        // });

        // function mouseout() {
        //     if (!fHover) return;
        //     map.getCanvasContainer().style.cursor = "default";
        //     map.setFeatureState(
        //         {
        //             source: fHover.source,
        //             sourceLayer: fHover.sourceLayer,
        //             id: fHover.id,
        //         },
        //         {
        //             hover: false,
        //         },
        //     );
        // }

        // function mouseover(feature) {
        //     fHover = feature;
        //     console.log(fHover.id);
        //     map.getCanvasContainer().style.cursor = "pointer";

        //     map.setFeatureState(
        //         {
        //             source: fHover.source,
        //             sourceLayer: fHover.sourceLayer,
        //             id: fHover.id,
        //         },
        //         {
        //             hover: true,
        //         },
        //     );
        // }

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
        .range(["white", "#d73027"]); // Adjust the range of colors as needed
</script>

<Scrolly bind:progress={timelineProgress}>
    {#each timelineData as item, index}
        <div class="info-wrapper" style="text-align: center;">
            <div class="info">
                <h1 class="date-heading">{item.date}</h1>
                <h2 class="event-heading">{item.event}</h2>
                <h3>{item.description}</h3>
                {#if item.image !== ""}
                    <img src={item.image} alt="img" />
                {/if}
                {#if item.link !== ""}
                    <a class="event-link" href={item.link}>Read More</a>
                {/if}
            </div>
        </div>
    {/each}

    <svelte:fragment slot="viz">
        <p class="timeline">{mapMaxtime}</p>
        <div id="viz2">
            <svg>
                {#each filteredBuildings as building}
                    {#key mapViewChanged}
                        <circle
                            {...getCoords(building)}
                            r="7"
                            fill={colorScale(
                                parseInt(
                                    building.TOTAL_VALUE.replace(/,/g, ""),
                                ),
                            )}
                            fill-opacity="0.7"
                            stroke="white"
                        />
                    {/key}
                {/each}
            </svg>
        </div>
    </svelte:fragment>
</Scrolly>

<style>
    @import url("$lib/global.css");
    #viz2 {
        flex: 1;
        min-height: 70vh;
        min-width: 50%;
    }
    #viz2 svg {
        position: absolute;
        z-index: 10;
        width: 100%;
        height: 100%;
        pointer-events: none;
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
