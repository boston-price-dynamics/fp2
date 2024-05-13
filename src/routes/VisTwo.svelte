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
    let mappings = [];
    let filteredBuildings;
    let mapMaxtime = 2013;
    let timelineProgress = 100;

    function getCoords(building) {
        let lat = building.lat;
        let long = building.long;
        let point = new mapboxgl.LngLat(long, lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }
    function getInfo(building, mappings, year) {
        const found = mappings.find(
            (b) =>
                b.ST_NUM === building.ST_NUM &&
                b.ST_NAME === building.ST_NAME &&
                parseInt(b.yr) === year
        );
        if (found) {
            return { id: found.mapboxid, percent: found.PercentChg };
        }
        return { id: -1 };
    }
    function removeDuplicates(arr) {
        const obj = {};

        for (let i = 0, len = arr.length; i < len; i++) {
            if (arr[i]["id"] >= 0) {
                obj[arr[i]["id"]] = arr[i];
            }
        }

        const newArr = new Array();

        for (const key in obj) {
            newArr.push(obj[key]);
        }

        return newArr;
    }
    let mapViewChanged = 0;
    $: map?.on("move", (evt) => mapViewChanged++);

    onMount(async () => {
        map = new mapboxgl.Map({
            container: "viz2",
            style: "mapbox://styles/jyoonsong/cluq04ktu05cr01qqb4rp4v4f",
            center: [-71.0841159, 42.3449576],
            zoom: 15.85,
            pitch: 60,
            // dragPan: false,
        });

        map.on("style.load", () => {
            // Insert the layer beneath any symbol layer.
            const layers = map.getStyle().layers;
            const labelLayerId = layers.find(
                (layer) => layer.type === "symbol" && layer.layout["text-field"]
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
                            [
                                "rgb",
                                ["feature-state", "redLevel"],
                                ["feature-state", "greenLevel"],
                                ["feature-state", "blueLevel"],
                            ],
                            "#ddd",
                        ],

                        // Use an 'interpolate' expression to
                        // add a smooth transition effect to
                        // the buildings as the user zooms in.
                        "fill-extrusion-height": [
                            "case",
                            [
                                "boolean",
                                ["feature-state", "heightChanged"],
                                false,
                            ],
                            [
                                "interpolate",
                                ["linear"],
                                ["feature-state", "heightLevel"],
                                0,
                                0,
                                1,
                                ["get", "height"],
                            ],
                            ["get", "height"],
                        ],
                    },
                },
                labelLayerId
            );
        });

        await new Promise((resolve) => map.on("load", resolve));

        buildings = await d3.csv("full_years.csv");
        mappings = await d3.csv("vistwo.csv");

        console.log(
            map.getFeatureState({
                source: "composite",
                sourceLayer: "building",
                id: 818117637,
            })
        );

        // let fHover;

        // map.on("mousemove", function (e) {
        //     var features = map.queryRenderedFeatures(e.point, {
        //         layers: ["add-3d-buildings"],
        //     });
        //     //we will change pointer and color for 42455719
        //     if (features[0]) {
        //         mouseover(features[0]);
        //     } else {
        //         mouseout();
        //     }
        // });

        // map.on("mouseout", function (e) {
        //     mouseout();
        // });

        // map.on("mouseout", function (e) {
        //     mouseout();
        // });

        // function mouseover(feature) {
        //     fHover = feature;
        //     // console.log(fHover.id);
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
    });

    $: timeScale = d3
        .scaleTime()
        .domain(d3.extent(timelineData, (evt) => evt.dt))
        .range([0, 100]);
    $: mapMaxtime = timeScale.invert(timelineProgress).getFullYear();

    $: {
        filteredBuildings = buildings.filter((building) => {
            return new Date(building.Year).getFullYear() === mapMaxtime;
        });

        const buildingInfos = buildings.map((b) =>
            getInfo(b, mappings, mapMaxtime)
        );
        const uniqueInfos = removeDuplicates(buildingInfos);
        if (uniqueInfos?.length > 0) {
            const allPercentages = mappings.map((m) =>
                parseFloat(m.PercentChg)
            );
            const minPercent = Math.min(...allPercentages);
            const maxPercent = Math.max(...allPercentages);

            // map.scrollZoom.disable();
            for (let buildingInfo of uniqueInfos) {
                const redRange = [255, 215];
                const greenRange = [255, 48];
                const blueRange = [255, 39];

                const srcMax = maxPercent - minPercent,
                    adjValue = buildingInfo.percent - minPercent;

                const redLevel = parseInt(
                    (adjValue * (redRange[1] - redRange[0])) / srcMax +
                        redRange[0]
                );
                const greenLevel = parseInt(
                    (adjValue * (greenRange[1] - greenRange[0])) / srcMax +
                        greenRange[0]
                );
                const blueLevel = parseInt(
                    (adjValue * (blueRange[1] - blueRange[0])) / srcMax +
                        blueRange[0]
                );

                map.setFeatureState(
                    {
                        source: "composite",
                        sourceLayer: "building",
                        id: buildingInfo.id,
                    },
                    {
                        hover: true,
                        isMarked: true,
                        redLevel: redLevel,
                        greenLevel: greenLevel,
                        blueLevel: blueLevel,
                    }
                );
            }

            // one dalton
            const heightRange = [0, 1];

            const srcMax = 2019 - 2010,
                adjValue = mapMaxtime - 2010;
            const heightLevel = parseInt(
                (adjValue * (heightRange[1] - heightRange[0])) / srcMax
            );

            map.setFeatureState(
                {
                    source: "composite",
                    sourceLayer: "building",
                    id: 818117637,
                },
                {
                    hover: true,
                    heightChanged: true,
                    redLevel: 255,
                    greenLevel: 39,
                    blueLevel: 255,
                    heightLevel: heightLevel, // reactive from 2017 to 2019
                }
            );
        }
    }
    let colorScale = d3
        .scaleLinear()
        .domain([0, 3000000])
        .range(["white", "#d73027"]); // Adjust the range of colors as needed
    let radiusScale = d3.scaleLinear().domain([0, 5000000]).range([0, 25]);
</script>

<Scrolly bind:progress={timelineProgress} --scrolly-viz-width="500px">
    {#each timelineData as item, index}
        <div class="info-wrapper" style="text-align: center;">
            <div class="info">
                <h1 class="date-heading">{item.year}</h1>
                <h2 class="event-heading">{item.event}</h2>
                <p>{item.description}</p>
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
        <!-- <p class="timeline">{mapMaxtime}</p> -->
        <div id="viz2">
            <!-- <svg>
                {#each filteredBuildings as building}
                    {#key mapViewChanged}
                        <text>{getId(building, mappings)}</text>
                        <circle
                            {...getCoords(building)}
                            r={radiusScale(
                                parseInt(building.TOTAL_VALUE.replace(/,/g, ""))
                            )}
                            fill={colorScale(
                                parseInt(building.TOTAL_VALUE.replace(/,/g, ""))
                            )}
                            fill-opacity="0.7"
                            stroke="white"
                            opacity="60%"
                        />
                    {/key}
                {/each}
            </svg> -->
        </div>
        <div id="map_legend">
            <div>Property value:</div>
            {#each [0, 500000, 1000000, 1500000, 2000000, 2500000, 3000000] as color, index}
                <div style="--color: {colorScale(color)}">${color}</div>
            {/each}
        </div>
    </svelte:fragment>
</Scrolly>

<style>
    @import url("$lib/global.css");
    #viz2 {
        /* flex: 1; */
        min-height: 70vh;
        /* min-width: 50%; */
        flex: 1;
        width: 500px;
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
        font-size: 70px; /* You can adjust the font size as needed */
        margin-top: 30px;
        margin-bottom: 0;
    }

    img {
        max-width: 50%;
        min-width: 40%;
    }

    #map circle {
        pointer-events: auto;
    }

    .info p {
        text-align: end;
    }

    .info {
        display: flex;
        flex-direction: column;
        align-items: end;
        padding-bottom: 2rem;
        gap: 1rem;
    }

    .info h2 {
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
