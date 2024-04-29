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
            scale = top >= half ? 2 : (half + top) / half;
            scaleText = (half + top / 2) / half;
        });
    });
</script>

<main>
    <div class="title">
        <!-- <p>
      
    </p> -->
        <h1 style={`transform: scale(${scaleText});`}>
            Boston's available housing stock is shrinking and becoming pricier.
            <br />
            <span class="yellow"
                >But instead of affordable housing, developers are choosing to
                target the <b>wealthy</b>.</span
            ><br />
            <span class="darkyellow"
                >How has focusing on luxury developments impacted housing
                prices?</span
            >
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
    <div class="intro">
        <h3>
            <b
                >The Greater Boston area is, undeniably, undergoing a housing
                crisis.</b
            >
            Bostonians are finding it harder and harder to find available - and affordable
            - housing, forcing some to leave the area entirely. Despite this, the
            area is simultaneously undergoing a sort of housing boom, with new high-rises
            redefining the city's skyline. <br /><br />So, Boston
            <em>is</em>
            building new housing.
            <b
                >But who is it actually for? What fraction of new developments
                can <em>you</em> afford?</b
            >
        </h3>
    </div>

    <VisOne />

    <!-- <LoremIpsum /> -->
    <div class="intro">
        <h3>
            <b>Evidently, developers overwhelmingly target wealthy buyers.</b>
            Much fewer affordable units were built in the same timeframe. However,
            some argue that any new housing is good housing. The argument is that
            by increasing the housing supply (even with only luxury housing), savings
            would eventually trickle down to all homebuyers.
            <b
                >But, is this true? How have luxury developments in Boston
                affected house prices in their surrounding neighborhoods?</b
            >
        </h3>
    </div>
</div>

<style>
    @import url("$lib/global.css");
</style>
