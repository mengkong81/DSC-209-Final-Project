<script>
  import * as d3 from 'd3';
  import { onMount } from 'svelte';

  let width = 1700;
  let height = 1000;
  let data, geoJson, selectedDate;
  let dateRange = [];
  const colorScale = d3.scaleThreshold()
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
      .scale(1800)
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
      .attr('fill', d => (d.properties.median_price !== null ? colorScale(d.properties.median_price) : '#ccc'))
      .attr('stroke', '#333')
      .attr('stroke-width', 0.5)
      .on('mouseover', function (event, d) {
        if (d.properties.median_price !== null) {
          const formattedPrice = d.properties.median_price.toLocaleString();
          d3.select(this)
            .transition()
            .duration(200)
            .attr('fill', d3.color(colorScale(d.properties.median_price)).brighter(0.5))
            .attr('stroke-width', 2);

          d3.select('#tooltip')
            .style('opacity', 1)
            .html(`${d.properties.NAME}: $${formattedPrice}`)
            .style('left', `${event.pageX + 10}px`)
            .style('top', `${event.pageY + 10}px`);
        }
      })
      .on('mouseout', function (event, d) {
        d3.select(this)
          .transition()
          .duration(200)
          .attr('fill', d => (d.properties.median_price !== null ? colorScale(d.properties.median_price) : '#ccc'))
          .attr('stroke-width', 0.5);

        d3.select('#tooltip').style('opacity', 0);
      });

    addLegend(svg, colorScale);
  }

  function createSlider() {
    const slider = d3.select('#slider');
    slider.append('h3').text('Select Timeline (Year)').style('font-size', '20px').style('margin-bottom', '10px');

    const yearRange = dateRange.map(d => d.split('/')[2]);

    const sliderScale = d3
      .scaleLinear()
      .domain([0, yearRange.length - 1])
      .range([0, 800]);

    const axis = d3.axisBottom(sliderScale).ticks(10).tickFormat(i => yearRange[i]);

    const svg = slider.append('svg').attr('width', 850).attr('height', 80);

    svg.append('g')
      .attr('transform', 'translate(25,50)')
      .call(axis)
      .selectAll('text')
      .style('font-size', '18px')
      .style('font-weight', 'bold')
      .style('fill', '#333');

    const handle = svg
      .append('circle')
      .attr('cx', 25)
      .attr('cy', 50)
      .attr('r', 10)
      .attr('fill', 'orange')
      .call(d3.drag().on('drag', event => {
        let x = Math.min(800, Math.max(0, event.x - 25));
        handle.attr('cx', x + 25);
        const index = Math.round(sliderScale.invert(x));
        selectedDate = dateRange[index];
        drawMap();
      }));
  }

  function addLegend(svg, colorScale) {
    const legendWidth = 800;
    const legendHeight = 20;

    const legend = svg.append('g')
      .attr('id', 'legend')
      .attr('transform', `translate(${(width - legendWidth) / 2}, 50)`);

    const axisScale = d3.scaleLinear()
      .domain([0, 1000000])
      .range([0, legendWidth]);

    const legendAxis = d3.axisBottom(axisScale)
      .tickValues([0, 100000, 300000, 500000, 750000, 1000000])
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
        { offset: '100%', color: '#084594' },
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

  function toggleAnswer(id) {
  const answer = document.getElementById(id);
  answer.classList.toggle('show');
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

<div style="
  margin-top: 20px; 
  padding: 10px; 
  line-height: 1.5; 
  max-width: 800px; 
  margin-left: auto; 
  margin-right: auto; 
  text-align: justify; 
  font-size: 20px; 
  color: #333; 
  text-indent: 40px;">
  <p>
    <strong>Insightful Visualization:</strong> Explore the evolution of U.S. housing market trends with this interactive visualization. Seamlessly hover over each state to reveal its median home price for the selected year. Navigate through two decades of data using the timeline slider and uncover regional trends. The dynamic color gradient highlights price variations, with deeper hues signifying higher housing values, offering a clear, data-driven perspective on the market.
  </p>
</div>

<div style="margin: 20px auto; max-width: 800px; font-family: Arial, sans-serif; line-height: 1.6;">
  <h2 style="text-align: center;">Project Insights</h2>

  <div style="border: 1px solid #ddd; border-radius: 5px; padding: 15px; margin-bottom: 15px;">
    <h3 style="margin: 0; font-size: 18px;">What have you done so far?</h3>
    <p style="margin-top: 10px;">
      I have set up the web page structure, ensuring a responsive and user-friendly layout, and added interactive features with Svelte and D3.js. The dataset is cleaned and prepared for visualization, and a deployment strategy using GitHub Pages is finalized. I’ve also tested components for smooth interactivity and created documentation to maintain clarity as the project progresses.
    </p>
  </div>

  <div style="border: 1px solid #ddd; border-radius: 5px; padding: 15px; margin-bottom: 15px;">
    <h3 style="margin: 0; font-size: 18px;">What will be the most challenging of your project to design and why?</h3>
    <p style="margin-top: 10px;">
      The most challenging aspects will be implementing advanced interactions like panning and zooming, ensuring intuitive functionality while maintaining performance with large datasets. Balancing interactivity with usability and adhering to accessibility standards adds complexity. Debugging across browsers and devices will also require significant effort to ensure consistency and reliability.
    </p>
  </div>
</div>


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
