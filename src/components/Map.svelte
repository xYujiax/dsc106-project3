<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let tooltip;
  let countriesData = [];
  let selectedYear = 1900;

  onMount(async() => {
    const svg = d3.select('#map')
      .attr('viewBox', `0 0 900 400`);

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
      .attr('fill', 'red')
      .attr('d', d3.geoPath().projection(projection))
      .attr('vector-effect', 'non-scaling-stroke')
      .style('stroke', 'black')
      .style('stroke-linejoin', 'round')
      .style('opacity', 1);

    // choropleth color scale
    let colorScale = d3.scaleSequential(d3.interpolateBlues);

    // map update function
    function updateMap(year) {
      const dataForYear = countriesData.filter(c => c.year === year);
      const productionValues = dataForYear.map(d => d.coal_production).filter(d => d !== undefined && d !== null); // keep 0s, remove null/undefined

      // min val for coal production that is greater than 0
      const minValue = 1e-6;

      // replace 0 with minVal
      const adjustedProductionValues = productionValues.map(d => (d === 0 ? minValue : d));

      // log scale add range to colors
      const logScale = d3.scaleLog()
        .domain([Math.max(minValue, d3.min(adjustedProductionValues)), d3.max(adjustedProductionValues)])
        .range([0, 1]);

      colorScale.domain([0, 1]);

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

      const leftPosition = event.pageX - tooltipWidth / 2;
      const topPosition = event.pageY - tooltipHeight - 10;

      paths.style('opacity', 0.5);
      d3.select(this).style('opacity', 1);

      tooltip
        .html(`${countryName}<br>Coal Production: ${coalProduction}<br>Population: ${population}`) // take out <br>Population: ${population} if unwanted
        .style('left', `${leftPosition}px`)
        .style('top', `${topPosition}px`)
        .style('opacity', .9); // show tooltip
    })

    // hover out
    paths.on('mouseout', function(event, d) {
      d3.selectAll('.country').style('opacity', 1);

      tooltip.transition()
        .duration(1)
        .style('opacity', 0); // hide tooltip
    });
  });

</script>

<style>
  .slider-container {
    text-align: center;
    width: 50%;
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
  }

  #map {
    width: 95vw;
    height: 80vh;
    max-height: 900px; /* adjust first three for map size */
    margin: 0 auto 0px auto; /* bottom margin (slider space) */
  }

</style>

<div class="year-display">{selectedYear}</div>

<!-- map size -->
<svg id='map' viewBox='0 0 900 400'></svg>

<!-- slider -->
<div class="slider-container">
  <input type='range' id='yearSlider' min='1900' max='2022' value={selectedYear}/>
</div>

<div class="year-display">{selectedYear}</div>