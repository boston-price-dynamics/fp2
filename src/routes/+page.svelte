<script>
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import VisOne from "./VisOne.svelte";
    import LoremIpsum from "./LoremIpsum.svelte";

    let opacity = 1;
    let scale = 1;
    let scaleText = 1;

    onMount(async () => {
        document.addEventListener("scroll", (e) => {
            const half = window.innerHeight / 2;
            const top = window.scrollY;
            opacity = top >= half ? 0 : (half - top) / half;
            scale = (half + top) / half;
            scaleText = (half + top / 2) / half;
        });
    });
</script>

<main>
    <div class="title">
        <h1 style={`transform: scale(${scaleText});`}>
            <span class="darkyellow"
                >Boston <em>is</em> building new housing.
            </span><br />
            <span class="yellow">But who is it actually for?</span><br />
            What fraction of new developments can <em>you</em> afford?
        </h1>
        <img
            class="bg"
            src="background.jpg"
            style={`opacity: ${opacity}; transform: scale(${scale});`}
            alt="bg"
        />
    </div>
</main>
<div class="wrapper">
    <VisOne />
</div>
<LoremIpsum />

<style>
    @import url("$lib/global.css");
</style>
