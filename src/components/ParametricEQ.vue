<template>
  <div ref="chartContainer" class="eq-container"></div>
</template>

<script setup>
import { ref, onMounted, watch, computed, useTemplateRef } from 'vue';
import * as d3 from 'd3';
import { throttle } from '../utils/throttle';

const filters = defineModel({ required: true, type: Array });

const { selectedId } = defineProps({
  selectedId: { type: Number, required: true },
});

const chartContainerRef = useTemplateRef('chartContainer');

const width = ref(800);
const height = ref(400);
const isDragging = ref(false); // Flag to track drag state

// Use an internal copy of the data for smooth dragging without constant prop updates
const internalFilters = ref(JSON.parse(JSON.stringify(filters.value)));

const selectedFilter = computed(() =>
  internalFilters.value.find((f) => f.id === selectedId)
);

const throttledFiltersUpdate = throttle((value) => {
  filters.value = JSON.parse(JSON.stringify(value));
}, 50);

// When the parent data changes, update our internal copy
watch(
  filters,
  (newValue) => {
    // Only redraw if we are not actively dragging.
    if (isDragging.value) return;

    internalFilters.value = JSON.parse(JSON.stringify(newValue));
    drawChart();
  },
  { deep: true }
);

// When the selected filter changes, redraw the chart to show the new selection
watch(
  () => selectedId,
  () => {
    drawChart();
  }
);

// --- MATH & CALCULATION HELPERS ---
const SAMPLE_RATE = 48000; // A standard sample rate for audio processing

function getFilterGain(filter, freq) {
  switch (filter.type) {
    case 'peaking':
    case 'low-shelf':
    case 'high-shelf':
      // The logic for descriptive filters remains the same
      const G = filter.db;
      const f0 = filter.hz;
      const Q = filter.q;
      if (filter.type === 'peaking') {
        if (Q <= 0) return 0;
        const x = Math.log2(freq / f0) * Q;
        const gain = G / (1 + 4 * x * x);
        return G > 0 ? Math.max(0, gain) : Math.min(0, gain);
      } else if (filter.type === 'low-shelf') {
        const steepness = Q * 5;
        const normalizedFreq = (Math.log10(freq) - Math.log10(f0)) * steepness;
        const factor = 1 / (1 + Math.exp(normalizedFreq));
        return G * factor;
      } else {
        // high-shelf
        const steepness = Q * 5;
        const normalizedFreq = (Math.log10(freq) - Math.log10(f0)) * steepness;
        const factor = 1 / (1 + Math.exp(-normalizedFreq));
        return G * factor;
      }

    case 'custom': {
      // Calculate frequency response from biquad coefficients
      const { b0, b1, b2, a1, a2 } = filter;
      const w = (2 * Math.PI * freq) / SAMPLE_RATE;

      // Numerator
      const numReal = b0 + b1 * Math.cos(w) + b2 * Math.cos(2 * w);
      const numImag = -b1 * Math.sin(w) - b2 * Math.sin(2 * w);

      // Denominator (a0 is normalized to 1)
      const denReal = 1 + a1 * Math.cos(w) + a2 * Math.cos(2 * w);
      const denImag = -a1 * Math.sin(w) - a2 * Math.sin(2 * w);

      const magnitude =
        Math.sqrt(numReal ** 2 + numImag ** 2) /
        Math.sqrt(denReal ** 2 + denImag ** 2);

      if (magnitude === 0) return -Infinity; // Avoid log(0)
      return 20 * Math.log10(magnitude);
    }
    default:
      return 0;
  }
}

function getTotalGain(freq) {
  return internalFilters.value.reduce((total, filter) => {
    const gain = getFilterGain(filter, freq);
    // For visualization, simple addition of dB is acceptable.
    return total + gain;
  }, 0);
}

// --- D3 DRAWING LOGIC ---
function drawChart() {
  if (!chartContainerRef.value) return;

  const margin = { top: 60, right: 40, bottom: 30, left: 60 }; // Adjusted bottom margin
  width.value = chartContainerRef.value.clientWidth;
  const innerWidth = width.value - margin.left - margin.right;
  const innerHeight = height.value - margin.top - margin.bottom;

  d3.select(chartContainerRef.value).select('svg').remove();

  const svg = d3
    .select(chartContainerRef.value)
    .append('svg')
    .attr('width', width.value)
    .attr('height', height.value)
    .append('g')
    .attr('transform', `translate(${margin.left},${margin.top})`);

  const xScale = d3.scaleLog().domain([10, 20000]).range([0, innerWidth]);
  const yScale = d3.scaleLinear().domain([-25, 25]).range([innerHeight, 0]);

  // --- AXES ---
  const xAxis = d3
    .axisBottom(xScale)
    .tickValues([10, 100, 1000, 10000])
    .tickFormat((d) => (d >= 1000 ? `${d / 1000}k` : d))
    .tickSize(6) // Ticks point down from the 0 line
    .tickPadding(8);

  const yAxis = d3
    .axisLeft(yScale)
    .tickValues([-25, 25]) // Only show -25 and 25
    .tickFormat((d) => `${d > 0 ? '+' : ''}${d}dB`); // Add dB suffix

  // Position X-axis at the 0dB line and remove the axis line itself
  svg
    .append('g')
    .attr('class', 'x-axis')
    .attr('transform', `translate(0, ${yScale(0)})`)
    .call(xAxis)
    .call((g) => g.select('.domain').remove()); // Remove the horizontal axis line

  // Position Y-axis as normal and remove the axis line
  svg
    .append('g')
    .attr('class', 'y-axis')
    .call(yAxis)
    .call((g) => g.select('.domain').remove()); // Remove the vertical axis line

  // --- GRIDS ---
  svg
    .append('line')
    .attr('class', 'zero-line')
    .attr('x1', 0)
    .attr('x2', innerWidth)
    .attr('y1', yScale(0))
    .attr('y2', yScale(0));

  const lineGenerator = d3
    .line()
    .x((d) => xScale(d.freq))
    .y((d) => yScale(d.db))
    .curve(d3.curveBasis);
  const areaGenerator = d3
    .area()
    .x((d) => xScale(d.freq))
    .y0(yScale(0))
    .y1((d) => yScale(d.db))
    .curve(d3.curveBasis);

  // Create the connector line first so it's in the background
  const qConnectorLine = svg.append('line').attr('class', 'q-connector-line');

  const totalCurvePath = svg.append('path').attr('class', 'total-curve');
  const selectedAreaPath = svg.append('path').attr('class', 'selected-area');

  // Tooltip group (initially hidden)
  const tooltip = svg
    .append('g')
    .attr('class', 'drag-tooltip')
    .style('pointer-events', 'none'); // So it doesn't interfere with dragging

  tooltip.append('rect');
  tooltip.append('path').attr('class', 'tooltip-arrow'); // Add path for the arrow
  tooltip.append('text');

  function updateCurves() {
    const lineData = d3
      .range(0, innerWidth, 2)
      .map((x) => ({
        freq: xScale.invert(x),
        db: getTotalGain(xScale.invert(x)),
      }));
    totalCurvePath.datum(lineData).attr('d', lineGenerator);

    if (selectedFilter.value) {
      const sf = selectedFilter.value;
      const selectedLineData = d3
        .range(0, innerWidth, 2)
        .map((x) => ({
          freq: xScale.invert(x),
          db: getFilterGain(sf, xScale.invert(x)),
        }));
      selectedAreaPath.datum(selectedLineData).attr('d', areaGenerator);
    } else {
      selectedAreaPath.attr('d', null);
    }
  }

  function updateTooltip(lines, centerX, handleY, handleRadius) {
    const textEl = tooltip.select('text');
    const rectEl = tooltip.select('rect');
    const arrowEl = tooltip.select('.tooltip-arrow');
    const padding = 4;
    const arrowWidth = 10;
    const arrowHeight = 5;
    const gap = 6;

    // 1. Build the tooltip content
    textEl.selectAll('tspan').remove();
    lines.forEach((line, i) => {
      textEl
        .append('tspan')
        .attr('x', 0)
        .attr('dy', i === 0 ? 0 : '1.2em') // Relative dy for subsequent lines
        .text(line);
    });

    // 2. Get BBox and calculate dimensions
    const textBBox = textEl.node().getBBox();
    const rectWidth = textBBox.width + padding * 2;
    const rectHeight = textBBox.height + padding * 2;

    // 3. Position rect and text within the tooltip group
    // The text is already centered by text-anchor: middle.
    // We move the text down to create top padding.
    textEl.attr('y', padding);

    // Position the rect around the text's new conceptual space
    rectEl
      .attr('x', -rectWidth / 2)
      .attr('y', 0)
      .attr('width', rectWidth)
      .attr('height', rectHeight);

    // 4. Draw arrow
    const arrowPath = `M 0,${rectHeight} l ${-arrowWidth / 2},0 l ${
      arrowWidth / 2
    },${arrowHeight} l ${arrowWidth / 2},${-arrowHeight} Z`;
    arrowEl.attr('d', arrowPath);

    // 5. Position the whole tooltip group
    const tooltipHeightWithArrow = rectHeight + arrowHeight;
    const handleTopY = handleY - handleRadius;

    const finalX = centerX;
    const finalY = handleTopY - tooltipHeightWithArrow - gap;

    tooltip.attr('transform', `translate(${finalX}, ${finalY})`);
  }

  if (selectedFilter.value && selectedFilter.value.type !== 'custom') {
    const sf = selectedFilter.value;

    // Create groups for Q handles to hold the visible line and the touch area
    const qHandleGroups = svg
      .selectAll('.q-handle-group')
      .data([0, 1]) // 0 for left handle, 1 for right
      .enter()
      .append('g')
      .attr('class', 'q-handle-group');

    // Add the visible line
    qHandleGroups.append('line').attr('class', 'q-handle-visual');

    // Add the wider, invisible touch area
    const qHitAreas = qHandleGroups
      .append('rect')
      .attr('class', 'q-handle-hit-area')
      .attr('width', 24) // 24px wide touch target
      .attr('height', innerHeight)
      .attr('y', 0)
      .attr('fill', 'transparent');

    // Add the main handle LAST so it is drawn on top of the lines
    const mainHandle = svg
      .append('circle')
      .attr('class', 'drag-handle main-handle')
      .attr('r', 8);

    const handleDrag = d3
      .drag()
      .subject(function () {
        // Define the drag origin to prevent jumps
        const s = d3.select(this);
        return { x: s.attr('cx'), y: s.attr('cy') };
      })
      .on('start', () => {
        isDragging.value = true;
        tooltip.style('display', 'block');
      })
      .on('drag', (event) => {
        sf.hz = Math.max(20, Math.min(20000, xScale.invert(event.x)));

        const draggedDb = yScale.invert(event.y);

        if (sf.type === 'peaking') {
          sf.db = Math.max(-25, Math.min(25, draggedDb));
        } else {
          const fullGain = draggedDb * 2;
          sf.db = Math.max(-25, Math.min(25, fullGain));
        }

        updateHandles();
        updateCurves();

        const handleYValue = sf.type === 'peaking' ? sf.db : sf.db / 2;
        const handleXPos = xScale(sf.hz);
        const handleYPos = yScale(handleYValue);
        const handleRadius = mainHandle.attr('r');

        updateTooltip(
          [`${sf.hz.toFixed(0)} Hz`, `${sf.db.toFixed(1)} dB`],
          handleXPos,
          handleYPos,
          handleRadius
        );

        throttledFiltersUpdate(internalFilters.value);
      })
      .on('end', () => {
        isDragging.value = false;
        filters.value = internalFilters.value;
        tooltip.style('display', 'none');
      });

    const qDrag = d3
      .drag()
      .subject(function () {
        // 'this' is the <rect> element being dragged. Its x/y is the origin.
        const s = d3.select(this);
        return { x: s.attr('x'), y: s.attr('y') };
      })
      .on('start', () => {
        isDragging.value = true;
        tooltip.style('display', 'block');
      })
      .on('drag', (event, d) => {
        // 'd' is the datum (0 for left, 1 for right)
        // event.x is the new x-coordinate of the hit-area's top-left corner.
        // Add half the width to get the line's center.
        const handleFreq = xScale.invert(event.x + 12);

        // Calculate the symmetry factor based on which handle is dragged
        let symmetryFactor;
        if (d === 1) {
          // Right handle
          symmetryFactor = Math.max(1.001, handleFreq / sf.hz);
        } else {
          // Left handle
          symmetryFactor = Math.max(1.001, sf.hz / handleFreq);
        }

        // Calculate Q from the factor. This is the inverse of the positioning logic.
        const newQ = 1 / (symmetryFactor - 1);
        sf.q = Math.max(0.1, Math.min(20, newQ)); // Clamp Q value

        updateHandles();
        updateCurves();

        const handleYValue = sf.type === 'peaking' ? sf.db : sf.db / 2;
        const handleXPos = xScale(sf.hz);
        const handleYPos = yScale(handleYValue);
        const handleRadius = mainHandle.attr('r');

        // Pass value as an array for consistent formatting, and position relative to the main handle
        updateTooltip(
          [`Q: ${sf.q.toFixed(2)}`],
          handleXPos,
          handleYPos,
          handleRadius
        );
        throttledFiltersUpdate(internalFilters.value);
      })
      .on('end', () => {
        isDragging.value = false;
        filters.value = internalFilters.value;
        tooltip.style('display', 'none');
      });

    mainHandle.call(handleDrag);
    // Attach drag behavior directly to the hit areas
    qHitAreas.call(qDrag);

    function updateHandles() {
      let handleYValue;
      if (sf.type === 'peaking') {
        handleYValue = sf.db;
      } else {
        handleYValue = sf.db / 2;
      }
      const yPos = yScale(handleYValue);
      mainHandle.attr('cx', xScale(sf.hz)).attr('cy', yPos);

      // Use a mathematically reversible factor for positioning
      const symmetryFactor = 1 + 1 / sf.q;
      const qPositions = [sf.hz / symmetryFactor, sf.hz * symmetryFactor];

      qHandleGroups.each(function (d, i) {
        const group = d3.select(this);
        const xPos = xScale(qPositions[i]);

        // Update the visible line
        group
          .select('.q-handle-visual')
          .attr('x1', xPos)
          .attr('x2', xPos)
          .attr('y1', 0)
          .attr('y2', innerHeight);

        // Update the invisible hit area, centering it on the line
        group.select('.q-handle-hit-area').attr('x', xPos - 12); // (24px width / 2)
      });

      // Update the horizontal connector line
      qConnectorLine
        .attr('x1', xScale(qPositions[0]))
        .attr('x2', xScale(qPositions[1]))
        .attr('y1', yPos)
        .attr('y2', yPos);
    }

    updateHandles();
  }

  updateCurves();
}

onMounted(() => {
  drawChart();
  window.addEventListener('resize', drawChart);
});
</script>

<style>
.eq-container {
  width: 100%;
  height: 400px;
  background-color: white;
  user-select: none;
}

.eq-container .x-axis path,
.eq-container .y-axis path,
.eq-container .x-axis .tick line,
.eq-container .y-axis .tick line {
  stroke: #e5e7eb;
}

.eq-container .x-axis .tick text {
  fill: #6b7280;
  font-size: 0.75rem;
}

.eq-container .y-axis .tick text {
  fill: #6b7280;
  font-size: 0.75rem;
}

.eq-container .zero-line {
  stroke: #d1d5db;
  stroke-width: 1px;
  stroke-dasharray: 2, 2;
}

.eq-container .total-curve {
  fill: none;
  stroke: black;
  stroke-width: 2px;
}

.eq-container .selected-area {
  fill: rgba(0, 0, 0, 0.05);
}

.eq-container .drag-handle {
  cursor: move;
}

.eq-container .main-handle {
  fill: black;
  stroke: white;
  stroke-width: 2px;
}

/* Style the visible Q line */
.eq-container .q-handle-visual {
  stroke: rgba(0, 0, 0, 0.2);
  stroke-width: 2px;
  stroke-linecap: round;
  pointer-events: none; /* The line itself shouldn't capture events */
}

/* Style the invisible touch area */
.eq-container .q-handle-hit-area {
  cursor: ew-resize;
}

.eq-container .q-handle-group:hover .q-handle-visual {
  stroke: rgba(0, 0, 0, 0.4);
}

/* Style for the new horizontal connector line */
.eq-container .q-connector-line {
  stroke: #d1d5db;
  stroke-width: 1px;
}

/* Style for the new drag tooltip */
.eq-container .drag-tooltip {
  display: none;
}

.eq-container .drag-tooltip rect {
  fill: #111827;
  rx: 4;
  opacity: 0.8;
}

.eq-container .drag-tooltip text {
  fill: white;
  font-size: 0.875rem;
  text-anchor: middle;
  dominant-baseline: text-before-edge;
}

.eq-container .tooltip-arrow {
  fill: #111827;
  opacity: 0.8;
}
</style>
