<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let tooltip;

  onMount(() => {
    const svg = d3.select("#map")
      .attr("viewBox", `0 0 900 400`);

    const width = 900;
    const height = 400;

    const projection = d3.geoNaturalEarth1()
      .scale((width / (2 * Math.PI)))
      .translate([width / 2, height / 2]);

    tooltip = d3.select("body").append("div")
      .attr("class", "tooltip")
      .style("opacity", 0);

    // map elements
    d3.json("https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world.geojson").then(data => {  // <--- we can download file and store in static
      const paths = svg.append("g")
      .selectAll("path")
      .data(data.features)
      .enter().append("path")
      .attr("class", "country")
      .attr("fill", "black")
      .attr("d", d3.geoPath().projection(projection))
      .attr("vector-effect", "non-scaling-stroke")
      .style("stroke", "#fff")
      .style("stroke-linejoin", "round")
      .style("opacity", 1);
      
      // hover initiate
      paths.on("mousemove", function(event, d) {
        paths.style("opacity", 0.5);
        d3.select(this).style("opacity", 1);

        tooltip
        .html(d.properties.name)
        .style("left", (event.pageX + 10) + "px")
        .style("top", (event.pageY + 10) + "px")
        .style("opacity", .9); // show tooltip
      })

      // hover out
      .on("mouseout", function(event, d) {
      d3.selectAll(".country").style("opacity", 1);
      d3.select(this).classed("active", false);

      tooltip.transition()
        .duration(1)
        .style("opacity", 0); // hide tooltip
      });

    });
  });

</script>

<input type="range" id="yearSlider" min="1900" max="2022" value="1900" />

<style>
  .tooltip {
    position: absolute;
    text-align: center;
    padding: .5em;
    background: lightsteelblue;
    border: 0px;
    border-radius: 8px;
    pointer-events: none;
    font-size: 14px;
    color: black;
  }
</style>

<svg id="map" width="1200" height="800"></svg>