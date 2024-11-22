<script>
	import * as d3 from 'd3';
	import { onMount } from 'svelte';
  
	let width = 1700; // Expanded width
	let height = 1000; // Expanded height
	let data, geoJson, selectedDate;
	let dateRange = [];
	let colorScale = d3.scaleThreshold()
  .domain([0, 100000, 300000, 500000, 750000, 1000000])
  .range(['#f7fbff', '#deebf7', '#c6dbef', '#9ecae1', '#6baed6', '#084594']);

  
	onMount(async () => {
	  data = await d3.csv('/median_prices.csv');
	  geoJson = await d3.json('/us-states.json');
	  dateRange = data.columns.slice(8); // Extract the date range
	  selectedDate = dateRange[0]; // Default to the first date
	  drawMap();
	  createSlider();
	});
  
	function drawMap() {
  const svg = d3
    .select('#map')
    .attr('width', width)
    .attr('height', height);

  const projection = d3
    .geoAlbersUsa()
    .scale(1800) // Adjusted for expanded map
    .translate([width / 2, height / 2]);

  const path = d3.geoPath().projection(projection);

  geoJson.features.forEach(feature => {
    const stateData = data.find(d => d.RegionName === feature.properties.NAME);
    feature.properties.median_price = stateData ? Math.round(+stateData[selectedDate]) : null;
  });

  svg
    .selectAll('path')
    .data(geoJson.features)
    .join('path')
    .attr('d', path)
    .attr('fill', d => d.properties.median_price !== null ? colorScale(d.properties.median_price) : '#ccc')
    .attr('stroke', '#333')
    .attr('stroke-width', 0.5)
    .on('mouseover', function (event, d) {
      // Highlight the state
      d3.select(this)
        .transition()
        .duration(200)
        .attr('fill', '#ffcc00') // Change to a highlight color
        .attr('stroke-width', 2); // Thicker border

      // Show tooltip
      if (d.properties.median_price !== null) {
        const formattedPrice = d.properties.median_price.toLocaleString();
        d3.select('#tooltip')
          .style('opacity', 1)
          .html(`${d.properties.NAME}: $${formattedPrice}`)
          .style('left', `${event.pageX + 10}px`)
          .style('top', `${event.pageY + 10}px`);
      }
    })
    .on('mouseover', function (event, d) {
  if (d.properties.median_price !== null) {
    // Brighten the current color slightly
    const currentColor = d3.select(this).attr('fill');
    const brighterColor = d3.color(currentColor).brighter(0.5);

    d3.select(this)
      .transition()
      .duration(200)
      .attr('fill', brighterColor) // Brighten the current color
      .attr('stroke-width', 10); // Thicker border

    // Show tooltip
    const formattedPrice = d.properties.median_price.toLocaleString();
    d3.select('#tooltip')
      .style('opacity', 1)
      .html(`${d.properties.NAME}: $${formattedPrice}`)
      .style('left', `${event.pageX + 10}px`)
      .style('top', `${event.pageY + 10}px`);
  }
})
.on('mouseover', function (event, d) {
  if (d.properties.median_price !== null) {
    d3.select(this)
      .transition()
      .duration(200)
      .attr('stroke', '#333') // Add a darker border
      .attr('stroke-width', 2); // Thicker border for highlighting

  // Hide tooltip
  d3.select('#tooltip').style('opacity', 0);
});


  addLegend(svg, colorScale);
}

  
function createSlider() {
  const slider = d3.select('#slider');
  slider.append('h3').text('Select Timeline (Year)').style('font-size', '20px').style('margin-bottom', '10px');

  const yearRange = dateRange.map(d => d.split('/')[2]); // Extract years from the dates

  const sliderScale = d3
    .scaleLinear()
    .domain([0, yearRange.length - 1])
    .range([0, 800]); // Adjusted slider width

  const axis = d3.axisBottom(sliderScale).ticks(10).tickFormat(i => yearRange[i]);

  const svg = slider.append('svg').attr('width', 850).attr('height', 80);

  svg.append('g')
  .attr('transform', 'translate(25,50)')
  .call(axis)
  .selectAll('text')
  .style('font-size', '18px') // Increase font size here
  .style('font-weight', 'bold') // Optionally, make it bold for emphasis
  .style('fill', '#333'); // Optionally, set a darker text color


  const handle = svg
    .append('circle')
    .attr('cx', 25)
    .attr('cy', 50)
    .attr('r', 10) // Larger handle for better visibility
    .attr('fill', 'orange')
    .call(d3.drag().on('drag', event => {
      let x = Math.min(800, Math.max(0, event.x - 25));
      handle.attr('cx', x + 25);
      const index = Math.round(sliderScale.invert(x));
      selectedDate = dateRange[index]; // Use full date for data lookup
      drawMap();
    }));
} 
function addLegend(svg, colorScale) {
  const legendWidth = 800; // Adjust for wider legend
  const legendHeight = 20;

  const legend = svg.append('g')
    .attr('id', 'legend')
    .attr('transform', `translate(${(width - legendWidth) / 2}, 50)`);

  const axisScale = d3.scaleLinear()
    .domain([0, 1000000]) // Updated domain to include $1,000,000
    .range([0, legendWidth]);

  const legendAxis = d3.axisBottom(axisScale)
    .tickValues([0, 100000, 300000, 500000, 750000, 1000000]) // Add $1,000,000 to tick values
    .tickFormat(d => `$${d.toLocaleString()}`);

  const gradient = svg.append('defs')
    .append('linearGradient')
    .attr('id', 'gradient');

  gradient.selectAll('stop')
    .data([
      { offset: '0%', color: '#f7fbff' },
      { offset: '12%', color: '#deebf7' },
      { offset: '30%', color: '#c6dbef' },
      { offset: '45%', color: '#9ecae1' },
      { offset: '66.6%', color: '#6baed6' },
      { offset: '100%', color: '#041e42' }, // Much darker blue for $1,000,000
    ])
    .enter()
    .append('stop')
    .attr('offset', d => d.offset)
    .attr('stop-color', d => d.color);

  legend.append('rect')
    .attr('width', legendWidth)
    .attr('height', legendHeight)
    .style('fill', 'url(#gradient)');

  legend.append('g')
    .attr('transform', `translate(0, ${legendHeight})`)
    .call(legendAxis)
    .selectAll('text')
    .style('font-size', '18px')
    .style('font-weight', 'bold');
}
  </script>

<div style="text-align: center; margin-bottom: 20px;">
	<h1 style="font-size: 28px; font-weight: bold;">
		U.S Housing Market Insights: Median Prices by State from January 2000 to October 2024
	</h1>
</div>

  
  <div id="slider" style="margin: 30px;"></div>
  <svg id="map"></svg>
  <div id="tooltip"></div>
  
  <style>
	#map {
	  border: 1px solid #ccc;
	  margin: 20px auto;
	  display: block;
	}
  
	#tooltip {
	  position: absolute;
	  background: white;
	  padding: 10px;
	  border: 1px solid #ccc;
	  border-radius: 5px;
	  pointer-events: none;
	  opacity: 0;
	  transition: opacity 0.2s ease;
	  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
	}
  </style>
  