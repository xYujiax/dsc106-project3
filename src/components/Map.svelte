<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let tooltip;
  let countriesData = [];
  let selectedYear = 1900;

  onMount(async() => {
    const svg = d3.select('#map').append('svg')
    .attr('width', window.innerWidth)
    .attr('height', window.innerHeight);
    //.attr('viewBox', `0 0 900 400`);

    const res = await fetch('https://raw.githubusercontent.com/xYujiax/dsc106-project3/main/static/countries_refactored.csv');
    const csv = await res.text();
    countriesData = d3.csvParse(csv, d3.autoType);

    const width = 900;
    const height = 400;

    const projection = d3.geoNaturalEarth1()
      .scale((width / (2 * Math.PI)))
      .translate([width / 2, height / 2]);

    tooltip = d3.select('body').append('div')
      .attr('class', 'tooltip')
      .style('opacity', 0);

    const worldData = await d3.json('https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world.geojson');
    const paths = svg.append('g')
      .selectAll('path')
      .data(worldData.features)
      .enter().append('path')
      .attr('class', 'country')
      .attr('d', d3.geoPath().projection(projection))
      .attr('vector-effect', 'non-scaling-stroke')
      .style('stroke', 'black')
      .style('stroke-linejoin', 'round')
      .style('opacity', 1);

    // choropleth color scale
    /* am deleting this cuz we doing binned color scale */
    //let colorScale = d3.scaleSequential(d3.interpolateBlues);
    
    const colorScale = d3.scaleQuantize(d3.schemeBlues[5]); // gets 5 colors from schemaBlues for binned colors
  
    // map update function
    function updateMap(year) {
      const dataForYear = countriesData.filter(c => c.year === year);
      const productionValues = dataForYear.map(d => d.coal_production).filter(d => d !== undefined && d !== null); // keep 0s, remove null/undefined

      // min val for coal production that is greater than 0
      const minValue = 1e-6;

      // replace 0 with minVal
      const adjustedProductionValues = productionValues.map(d => (d === 0 ? minValue : d));
      console.log(adjustedProductionValues)

      // log scale add range to colors
      const logScale = d3.scaleLog()
        .domain([Math.max(minValue, d3.min(adjustedProductionValues)), d3.max(adjustedProductionValues)])
        .range([0, 1]);
      
      console.log(logScale.tickFormat(20, "$,f"))

      // Update color scale domain based on log-scaled values
      //colorScale.domain([d3.min(adjustedProductionValues), d3.max(adjustedProductionValues)]);
      //colorScale.domain([0, 1]);

      paths.each(function(d) {
        const countryName = d.properties.name;
        const countryData = dataForYear.find(c => c.country === countryName);
        const coalProduction = countryData ? countryData.coal_production : null;

        // apply log scale -> color
        // check null/undefnied/0, otherwise color
        let fillColor;
        if (coalProduction === null || coalProduction === undefined) {
          fillColor = 'gray'; // for entries with no data
        } else {
          fillColor = coalProduction === 0 ? colorScale(0) : colorScale(logScale(coalProduction));
        }
        d3.select(this).attr('fill', fillColor);
        console.log('colorScaleValue:', colorScale(logScale(coalProduction)));
      });
    }


    // call updateMap with current year
    updateMap(selectedYear);

    // slider event
    d3.select('#yearSlider').on('input', function(event) {
      selectedYear = +event.target.value;
      updateMap(selectedYear);
    });

    // hover initiate
    paths.on('mousemove', function(event, d) {
      const countryName = d.properties.name;
      const countryData = countriesData.find(c => c.country === countryName && c.year === selectedYear);
      const coalProduction = countryData ? countryData.coal_production : 'No data';
      const population = countryData ? countryData.population : 'No data';

      const tooltipWidth = tooltip.node().getBoundingClientRect().width;
      const tooltipHeight = tooltip.node().getBoundingClientRect().height;

      const leftPosition = event.pageX;
      const topPosition = event.pageY;

      paths.style('opacity', 0.5);
      d3.select(this).style('opacity', 1);

      tooltip
        .html(`${countryName}<br>Coal Production in Tera-watt Hours: ${coalProduction}<br>Population: ${population}`) // take out <br>Population: ${population} if unwanted
        .style('opacity', .9)
        .style('left', (event.clientX + 10) + 'px')
        .style('top', (event.clientY + 10) + 'px'); // show tooltip
    })

    // hover out
    paths.on('mouseout', function(event, d) {
      d3.selectAll('.country').style('opacity', 1);

      tooltip.transition()
        .duration(1)
        .style('opacity', 0); // hide tooltip
    });

    // LEGEND
    const legend = svg.append("g")
        .attr("class", "legend")
        .attr("transform", "translate(110, 210)"); 

    // reverse color scale range cuz rn legend is upside-down default
    const reversedColorRange = colorScale.range().reverse();

    // color rectangles for legend
    legend.selectAll("rect")
        .data(reversedColorRange)
        .enter().append("rect")
        .attr("x", 0) // rectangles start at the same x-coordinate
        .attr("y", (d, i) => i * 15) // spacing between rectangles
        .attr("width", 10) // width of each rect
        .attr("height", 10) // height
        .attr("fill", d => d);

    // text labels for legend
    legend.selectAll("text")
        .data(reversedColorRange)
        .enter().append("text")
        .attr("x", 15) // text positioning to the right of the rect
        .attr("y", (d, i) => i * 15 + 8) // vert position
        .attr("text-anchor", "start") // align to text-element strt
        .text((d, i) => {
            // show range corresponding to each color
            const domain = colorScale.invertExtent(d);
            return `${domain[0].toFixed(2)} - ${domain[1].toFixed(2)}`;
        });

    // legend title
    legend.append("text")
        .attr("x", 0)
        .attr("y", -10) // Adjust the vertical position of the title
        .attr("text-anchor", "start")
        .style("font-weight", "bold")
        .text("Coal Production (log-scaled)"); // Replace with your desired title

  });

</script>

<!-- throwing html here. ik it's eggregious but too lazy to put in another doc -->
  <h1>Which countries wanna give you more lung cancer every year?</h1>

  <!-- slider -->
  <div class="slider-container">
    <input type='range' id='yearSlider' min='1900' max='2022' value={selectedYear}/>
  </div>
  <div class="year-display">Year: {selectedYear}</div>

<!-- map size -->
<svg id='map' viewBox='0 0 900 400'></svg>

<a href="https://raw.githubusercontent.com/xYujiax/dsc106-project3/main/%20Writeup.txt">Write-up</a>

<style>

@import url('https://fonts.googleapis.com/css2?family=Nunito:wght@300;400;700&display=swap');

    :root {
        --color-bg: /* Add a background color for the to-do app! */ ex: #f7f7f7;
        --color-outline: /* Add an outline color for the to-do app! */ ex: #c2c2c2;
        --color-shadow: /* Add a shadow for the to-do app! */ ex: hsl(0, 0%, 0%, 0.1);
        --color-text: /* Add a color for the text! */ ex: #222222;
    }

    *,
    *::before,
    *::after {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
    }

    h1{
      text-align: center;
      font-family: 'Nunito', sans-serif;
      font-weight: bold;
      line-height: 2;
      font-size: 48px;
      color: var(--color-text);
  }

  .slider-container {
    text-align: center;
    width: 55%;
    margin: 20px auto;
  }

  #yearSlider {
    width: 100%; /* adjust this to change 'width' of slider */
    margin: 0 auto;
  }

  .year-display {
    text-align: center;
    font-size: 32px;
    margin: 10px auto 20px auto;
    font-family: 'Nunito', sans-serif;
  }

  #map { 
    margin: 0 auto 0px auto; /* bottom margin (slider space) */
    overflow: hidden;
  }

  :global(.tooltip) {
    position: absolute;
    pointer-events: none;
    background: rgba(255, 255, 255, 0.8); /* semi-transparent white background */
    border: 1px solid #ddd; /* light grey border */
    padding: 5px 10px; /* some padding inside the tooltip */
    border-radius: 4px; /* rounded corners */
    text-align: left; /* align text to left */
    font-size: 2em; /* adjust font size as needed */
    font-family: 'Nunito', sans-serif;
    line-height: 1.4; /* line spacing for better readability */
    color: #333; /* and a darker text color for contrast */
  }

:global(.legend rect) {
    width: px; /* Adjust the width of the rectangles */
    height: px; /* Adjust the height of the rectangles */
}

:global(.legend text) {
  font-family: 'Nunito', sans-serif; /* Adjust the font size of the text */
  font-size: 9px; 
}


</style>



