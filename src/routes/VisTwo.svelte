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
    let minPercent = -1;
    let maxPercent = 1;
    let colorScale;

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
                parseInt(b.yr) === year,
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
            dragPan: false,
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
                            [
                                "boolean",
                                ["feature-state", "colorChanged"],
                                false,
                            ],
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

                        "fill-extrusion-opacity": 0.2,
                    },
                },
                labelLayerId,
            );
            map.addLayer(
                {
                    id: "add-3d-buildings-1",
                    source: "composite",
                    "source-layer": "building",
                    filter: ["==", "extrude", "true"],
                    type: "fill-extrusion",
                    minzoom: 15,
                    paint: {
                        "fill-extrusion-color": [
                            "case",
                            [
                                "boolean",
                                ["feature-state", "colorChanged"],
                                false,
                            ],
                            ["feature-state", "color"],
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
                            0,
                        ],

                        "fill-extrusion-opacity": 1,
                    },
                },
                labelLayerId,
            );
        });

        await new Promise((resolve) => map.on("load", resolve));

        // buildings = await d3.csv("full_years.csv");
        buildings = await d3.csv("full_years_new.csv");
        mappings = await d3.csv("vistwo.csv");

        map.scrollZoom.disable();

        // console.log(
        //     map.getFeatureState({
        //         source: "composite",
        //         sourceLayer: "building",
        //         id: 818117637,
        //     }),
        // );

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

        const allPercentages = mappings.map((m) => parseFloat(m.PercentChg));
        minPercent = Math.min(...allPercentages);
        maxPercent = Math.max(...allPercentages);

        console.log(minPercent, maxPercent);
        colorScale = d3
            .scaleLinear()
            .domain([-30, 180])
            .range(["white", "#d73027"]); // Adjust the range of colors as needed

        const buildingInfos = buildings.map((b) =>
            getInfo(b, mappings, mapMaxtime),
        );
        const uniqueInfos = removeDuplicates(buildingInfos);
        if (uniqueInfos?.length > 0) {
            // map.scrollZoom.disable();
            for (let buildingInfo of uniqueInfos) {
                const color = colorScale(buildingInfo.percent * 100);

                map.setFeatureState(
                    {
                        source: "composite",
                        sourceLayer: "building",
                        id: buildingInfo.id,
                    },
                    {
                        colorChanged: true,
                        heightChanged: true,
                        isMarked: true,
                        color: color,
                        heightLevel: 1,
                    },
                );
            }

            // one dalton
            const heightRange = [0, 1];

            const srcMax = 71 - 43,
                adjValue = timelineProgress - 43;
            const heightLevel =
                (adjValue * (heightRange[1] - heightRange[0])) / srcMax;

            map.setFeatureState(
                {
                    source: "composite",
                    sourceLayer: "building",
                    id: 818117637,
                },
                {
                    colorChanged: true,
                    heightChanged: true,
                    color: "rgb(70, 70, 70)",
                    heightLevel: heightLevel, // reactive from 2017 to 2019
                },
            );

            for (let exception of [536526392, 29945243]) {
                map.setFeatureState(
                    {
                        source: "composite",
                        sourceLayer: "building",
                        id: exception,
                    },
                    {
                        colorChanged: false,
                        heightChanged: true,
                        heightLevel: 0, // reactive from 2017 to 2019
                    },
                );
            }
        }
    }

    console.log(minPercent, maxPercent);

    $: {
        buildings.sort((a, b) => new Date(a.Year) - new Date(b.Year));

        buildings.forEach((building) => {
            const currentYear = new Date(building.Year).getFullYear();
            const prevYear = currentYear - 1;
            if (prevYear >= 2010) {
                const filteredBuildingsPrevYear = buildings.filter(
                    (building) => {
                        return (
                            new Date(building.Year).getFullYear() === prevYear
                        );
                    },
                );

                const prevYearBuilding = filteredBuildingsPrevYear.find(
                    (prevBuilding) => {
                        return (
                            prevBuilding.ST_NUM === building.ST_NUM &&
                            prevBuilding.ST_NAME === building.ST_NAME
                        );
                    },
                );
                if (
                    prevYearBuilding !== undefined &&
                    parseInt(prevYearBuilding.TOTAL_VALUE.replace(/,/g, "")) >
                        0 &&
                    parseInt(building.TOTAL_VALUE.replace(/,/g, "")) > 0
                ) {
                    const percentChange =
                        ((parseInt(building.TOTAL_VALUE.replace(/,/g, "")) -
                            parseInt(
                                prevYearBuilding.TOTAL_VALUE.replace(/,/g, ""),
                            )) /
                            parseInt(
                                prevYearBuilding.TOTAL_VALUE.replace(/,/g, ""),
                            )) *
                        100;
                    building.percentChange = percentChange;
                } else {
                    building.percentChange = 0;
                }
            } else {
                building.percentChange = 0;
            }
        });
    }
    let radiusScale = d3.scaleLinear().domain([0, 20]).range([0, 25]);
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
                                building.percentChange
                            )}
                            fill={colorScale(
                                building.percentChange
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
            <div id="label">% Chg. in Property Value:</div>
            {#each [-30, 0, 30, 60, 90, 120, 150, 180] as color, index}
                <div style="--color: {colorScale(color)}">{color}%</div>
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

        & #label {
            flex: 2.5;
        }
    }
</style>
