<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";
    import Scrolly from "svelte-scrolly";

    let data = [];
    let progress = 1;

    function printDifference(data, column) {
        const currentYear =
            progress >= 90
                ? 2022
                : 2000 + parseInt(((progress + 10) / 100) * 22);
        const found = data.find((d) => parseInt(d.year) === currentYear);
        const baseline = data.find((d) => parseInt(d.year) === 2000);
        if (found && baseline) {
            const diff =
                ((found[column] - baseline[column]) / baseline[column]) * 100;
            return diff >= 0 ? "+" + diff.toFixed(2) : diff.toFixed(2);
        }
        return 0;
    }

    function handleLine(data, column) {
        var margin = { top: 10, right: 30, bottom: 30, left: 60 },
            width = 800 - margin.left - margin.right,
            height = 500 - margin.top - margin.bottom;

        var x = d3.scaleLinear().domain([1999, 2023]).range([0, width]);
        var y = d3
            .scaleLinear()
            .domain([
                0,
                d3.max(data, function (d) {
                    const val = parseFloat(
                        d["Mass. condos median sales price"]
                    );
                    return val;
                }),
            ])
            .range([height, 0]);
        var line = d3
            .line()
            .x(function (d, index, arr) {
                if (((progress + 10) / 100) * 22 < index) {
                    return undefined;
                }
                return margin.left + x(d.year);
            })
            .y(function (d, index, arr) {
                if (((progress + 10) / 100) * 22 < index) {
                    return undefined;
                }
                return margin.bottom + y(d[column]);
            });
        return line(data);
    }

    onMount(async () => {
        data = await d3.csv("data.csv");
        console.log(data);

        // // set the dimensions and margins of the graph
        // var margin = { top: 10, right: 30, bottom: 30, left: 60 },
        //     width = 800 - margin.left - margin.right,
        //     height = 500 - margin.top - margin.bottom;

        // // append the svg object to the body of the page
        // var svg = d3
        //     .select("#my_dataviz")
        //     .append("svg")
        //     .attr("width", width + margin.left + margin.right)
        //     .attr("height", height + margin.top + margin.bottom)
        //     .append("g")
        //     .attr(
        //         "transform",
        //         "translate(" + margin.left + "," + margin.top + ")"
        //     );

        // // Add X axis --> it is a date format
        // var x = d3.scaleLinear().domain([1999, 2023]).range([0, width]);
        // svg.append("g")
        //     .attr("transform", "translate(0," + height + ")")
        //     .call(d3.axisBottom(x).tickFormat((d) => d));

        // // Add Y axis
        // var y = d3
        //     .scaleLinear()
        //     .domain([
        //         0,
        //         d3.max(data, function (d) {
        //             const val = parseFloat(
        //                 d["Mass. condos median sales price"]
        //             );
        //             return val;
        //         }),
        //     ])
        //     .range([height, 0]);
        // svg.append("g").call(d3.axisLeft(y));
    });
</script>

<div id="viz3">
    <Scrolly bind:progress --scrolly-layout="viz-first" threshold="0">
        <svelte:fragment slot="viz">
            <!-- <div id="my_dataviz"></div> -->
            <svg id="svg" width="800" height="500"
                ><g transform="translate(60,10)"
                    ><g
                        transform="translate(0,460)"
                        fill="none"
                        font-size="10"
                        font-family="sans-serif"
                        text-anchor="middle"
                        ><path
                            class="domain"
                            stroke="currentColor"
                            d="M0,6V0H710V6"
                        ></path><g
                            class="tick"
                            opacity="1"
                            transform="translate(29.583333333333332,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(88.75,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2002</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(147.91666666666669,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2004</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(207.08333333333334,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2006</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(266.25,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2008</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(325.41666666666663,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2010</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(384.5833333333333,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2012</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(443.75,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2014</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(502.9166666666667,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2016</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(562.0833333333333,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2018</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(621.25,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2020</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(680.4166666666667,0)"
                            ><line stroke="currentColor" y2="6"></line><text
                                fill="currentColor"
                                y="9"
                                dy="0.71em">2022</text
                            ></g
                        ></g
                    ><g
                        fill="none"
                        font-size="10"
                        font-family="sans-serif"
                        text-anchor="end"
                        ><path
                            class="domain"
                            stroke="currentColor"
                            d="M-6,460H0V0H-6"
                        ></path><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,460)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">0</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,420.95076400679113)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">50,000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,381.9015280135823)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">100,000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,342.8522920203735)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">150,000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,303.8030560271647)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">200,000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,264.75382003395583)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">250,000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,225.70458404074702)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">300,000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,186.6553480475382)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">350,000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,147.60611205432937)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">400,000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,108.55687606112055)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">450,000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,69.50764006791171)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">500,000</text
                            ></g
                        ><g
                            class="tick"
                            opacity="1"
                            transform="translate(0,30.458404074702866)"
                            ><line stroke="currentColor" x2="-6"></line><text
                                fill="currentColor"
                                x="-9"
                                dy="0.32em">550,000</text
                            ></g
                        ></g
                    ></g
                >
                {#key data}
                    <path
                        id="income"
                        fill="none"
                        stroke="#1a9850"
                        stroke-width="3"
                        d={handleLine(data, "Mass. median household income")}
                    ></path>
                    <path
                        id="housing"
                        fill="none"
                        stroke="#d73027"
                        stroke-width="3"
                        d={handleLine(data, "Mass. condos median sales price")}
                    ></path>
                {/key}
            </svg>
            <div class="label">
                Percent change in Income <br />from 2000 to {progress >= 90
                    ? 2022
                    : 2000 + parseInt(((progress + 10) / 100) * 22)}<br />
                <strong
                    >{printDifference(
                        data,
                        "Mass. condos median sales price"
                    )}%</strong
                >
            </div>
            <div class="label">
                Percent change in House Price<br />from 2000 to {progress >= 90
                    ? 2022
                    : 2000 + parseInt(((progress + 10) / 100) * 22)}<br />
                <strong
                    >{printDifference(
                        data,
                        "Mass. median household income"
                    )}%</strong
                >
            </div>
        </svelte:fragment>
        <div style="height: 2000px"></div>
    </Scrolly>
</div>

<style>
    @import url("$lib/global.css");
</style>
