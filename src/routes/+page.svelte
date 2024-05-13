<script>
  import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  import { onMount } from "svelte";
  import VisOne from "./VisOne.svelte";
  import VisThree from "./VisThree.svelte";
  import LoremIpsum from "./LoremIpsum.svelte";
  import VisTwo from "./VisTwo.svelte";

  let opacity = 1;
  let scale = 1;
  let scaleText = 1;
  let incomeFilter = 100000;

  let USDollar = new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
    maximumFractionDigits: 0,
  });

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
        >But instead of affordable housing, developers are choosing to target
        the <b>wealthy</b>.</span
      ><br />
      <span class="darkyellow"
        >How has focusing on luxury developments impacted housing prices?</span
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
      <b>The Greater Boston area is, undeniably, undergoing a housing crisis.</b
      >
      Bostonians are finding it harder and harder to find available - and affordable
      - housing, forcing some to leave the area entirely. Despite this, the area
      is simultaneously undergoing a sort of housing boom, with new high-rises redefining
      the city's skyline. <br /><br />So, Boston
      <em>is</em>
      building new housing.
      <b
        >But who is it actually for? What fraction of new developments can <em
          >you</em
        > afford?</b
      >
      In the ensuing visualization, we define an "affordable" unit based on the standard
      rule of spending no more than 30% of your income on housing costs. Based on
      current mortgage rates, a house is affordable if it costs no more than ~3.3x
      your annual household income.
    </h3>
    <label id="income">
      Enter your annual household income:
      <div>
        <input type="range" min="0" max="1000000" bind:value={incomeFilter} />
        <year>{USDollar.format(incomeFilter)}</year>
      </div>
    </label>
  </div>

  <VisOne bind:incomeFilter />

  <!-- <LoremIpsum /> -->
  <div class="content">
    <h3>
      <b>Evidently, developers overwhelmingly target wealthy buyers.</b>
      Much fewer affordable units were built in the same timeframe. However, some
      argue that any new housing is good housing. The argument is that by increasing
      the housing supply (even with only luxury housing), savings would eventually
      trickle down to all homebuyers.<br /><br />
      <b
        >But, is this true? How have luxury developments in Boston affected
        house prices in their surrounding neighborhoods?</b
      >
      To understand this, let's look at a case study of One Dalton, a 850,000 sq
      ft skyscraper in Boston. It is the tallest residential building in New England,
      home to the Four Seasons Hotel and private residences. Scroll through the ensuing
      visualization to see how its construction has affected the property values
      of the surrounding homes.
    </h3>
  </div>

  <div>
    <h3></h3>
  </div>

  <VisTwo />

  <div class="content">
    <h3>
      <b>
        It's clear from these case studies that the construction of luxury
        high-rises is <em>not</em> the answer to addressing Boston's housing crisis.</b
      >
      Neighboring house prices continue to rise during and after their development.
      We find no evidence of a trickle-down effect from the (modest) increase in
      housing supply resulting from luxury developments.
      <br /><br />
      To further emphasize the dire nature of the situation,
      <b
        >let's see just how much
        <em>faster</em> housing prices are rising relative to household incomes</b
      > - the true measure of affordability.
    </h3>
  </div>
  <VisThree />

  <div class="content">
    <h3>
      In summary, our analysis shows that<br />
      1) In the past few decades, Boston developers are constructing
      <b>only for the rich.</b><br />
      2)
      <b
        >Focusing on luxury developments alone does not benefit housing
        affordability</b
      >; rather, neighboring prices continue to rise during and after their
      construction.<br />
      3) As a result,
      <b>home prices are rapidly outpacing household incomes</b>, worsening the
      affordability crisis.<br /><br />
      More housing - and specifically, more affordable housing - is critically needed.
    </h3>
  </div>

  <div class="references">
    <h3>
      <b>Authored by: Jerry Li, Jaeyoon Song, Riley Oh, Consecrata Rozario </b>
      <br />
      This project was developed with guidance and feedback from the
      <a href="https://www.mapc.org/"
        >Metropolitan Area Planning Commission (MAPC)</a
      >.<br />
      <br />
      <b
        >In this interactive visualization we used the following data sources:</b
      ><br />

      <a href="https://data.boston.gov/dataset/parcels-2023"
        >City of Boston parcels, 2023</a
      ><br />
      <a href="https://data.boston.gov/dataset/property-assessment"
        >Boston property assesments, 2010-2024
      </a><br />
      <a href="https://data.census.gov/">American Community Survey, 2022</a><br
      />

      <br />
      The second visualization was inspired by the Boston Globe's
      <a
        href="https://apps.bostonglobe.com/2023/10/special-projects/spotlight-boston-housing/boston-towers-of-wealth/"
      >
        Reckoning with Boston's Towers of Wealth</a
      >.<br />
      <a href="https://en.wikipedia.org/wiki/One_Dalton">Wiki: One Dalton</a><br
      />
      <a
        href="https://www.bostonplans.org/projects/development-projects/1-dalton-street"
        >Boston Planning and Development Agency One Dalton Street records</a
      >
      <br /><br />
      We would also like to thank and acknowledge MIT's 6.C35 instructors and TAs
      for their instruction and guidance with this project.
    </h3>
  </div>
</div>

<style>
  @import url("$lib/global.css");
  #income {
    margin-left: auto;
    margin-right: auto;
    display: flex;
    flex-direction: column;
    font-size: 18px;

    & div {
      display: flex;
      justify-content: space-between;

      & year {
        font-size: 24px;
      }
    }
  }
</style>
