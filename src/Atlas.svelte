<script>
  import { onMount } from 'svelte';
  import { mapViewConfigStore, sideStore, instanceIDStore,
    attentionHeadColorStore, tooltipConfigStore, mapHeadStore } from './store';
  import { createEventDispatcher } from 'svelte';
  import * as d3 from 'd3';

  export let attentionDataDir;
  export let saliencyDataFilepath;
  export let atlasDataFilepath;

  let svg = null;
  let atlasData = null;
  let attentions = null;
  let saliencies = null;
  let tokenSize = null;

  // Tooltip variables
  let tooltipConfig = null;
  tooltipConfigStore.subscribe(value => {tooltipConfig = value;});

  let sideInfo = null;

  let viewContainer = null;
  let mapHead = {layer: 9, head: 8};
  let curLayer = 9;
  let curHead = 8;

  const red = d3.hcl(23, 85, 56);
  const purple = d3.hcl(328, 85, 56);
  const blue = d3.hcl(274, 85, 56);

  let instanceID = 1562;

  const dispatch = createEventDispatcher();
  let isShown = true;

  let SVGWidth = 800;
  let SVGHeight = 800;

  let instanceViewConfig = undefined;

  const SVGPadding = {top: 40, left: 10, right: 10, bottom: 3};

  const ease = d3.easeCubicInOut;

  const round = (num, decimal) => {
    return Math.round((num + Number.EPSILON) * (10 ** decimal)) / (10 ** decimal);
  };

  const padZeroLeft = (num, digit) => {
    return Array(Math.max(digit - String(num).length + 1, 0)).join(0) + num;
  };

  const createGraph = () => {

    const layerNum = attentions.length;
    const headNum = attentions[0].length;
    const layerNameWidth = 47;
    const headNameHeight = 20;

    // console.log(SVGWidth, SVGHeight);

    let availableWidth = SVGWidth - 210 - layerNameWidth - SVGPadding.left - SVGPadding.right;
    let availableHeight = SVGHeight - SVGPadding.top - SVGPadding.bottom - headNameHeight;

    let availableLength = Math.min(availableHeight, availableWidth);
    // console.log(SVGHeight, availableLength, availableWidth, availableHeight);
    const gridGap = 8;

    const gridLength = Math.floor((availableHeight - (layerNum - 1) * gridGap) / layerNum);
    const maxOutRadius = gridLength / 2;
    const minOutRadius = 10;

    let adjustedRowGap = Math.floor((availableWidth - maxOutRadius - headNum * gridLength) / (layerNum - 1));
    let adjustedColGap = Math.floor((availableHeight - layerNum * gridLength) / (layerNum - 1));

    svg = d3.select('.atlas-svg-full')
      .attr('viewbox', `0 0 ${availableWidth + layerNameWidth} ${availableHeight}`)
      .attr('width', availableWidth + layerNameWidth)
      .attr('height', availableHeight + headNameHeight);

    // Add a border
    svg.append('rect')
      .attr('class', 'border-rect')
      .attr('width', availableLength)
      .attr('height', availableLength)
      .style('stroke', 'black')
      .style('fill', 'none');

    let donutGroup = svg.append('g')
      .attr('class', 'donut-group')
      .attr('transform', `translate(${SVGPadding.left + maxOutRadius + layerNameWidth},
        ${maxOutRadius + headNameHeight})`);
    
    // Create color scale
    let hueScale = d3.scaleLinear()
      .domain([-1, 0, 1])
      .range([red, purple, blue]);

    let lightnessScale = d3.scaleLinear()
      .domain([0, 1])
      .range([130, 40]);

    // Use square root scale
    let outRadiusScale = d3.scaleLinear()
      .domain([0, 1])
      .range([minOutRadius, maxOutRadius]);

    let ringRadiusScale = d3.scaleLinear()
      .domain([0, 1])
      .range([4, 7]);

    let scales = {
      hueScale: hueScale,
      lightnessScale: lightnessScale,
      outRadiusScale: outRadiusScale,
      ringRadiusScale: ringRadiusScale
    };

    let donuts = donutGroup.selectAll('g.donut')
      .data(atlasData)
      .join('g')
      .attr('class', 'donut')
      .attr('transform', d => `translate(${d.head * (maxOutRadius * 2 + adjustedRowGap)},
        ${(layerNum - d.layer - 1) * (maxOutRadius * 2 + adjustedColGap)})`)
      .style('pointer-events', 'fill')
      .style('cursor', 'pointer');

    // Draw the donuts
    donuts.each((d, i, g) => drawDonut(d, i, g, scales));

    donuts.on('mouseover',
      (e, d) => {
        // Show the tooltip
        let node = e.currentTarget;
        let position = node.getBoundingClientRect();
        let curWidth = position.right - position.left;

        let tooltipCenterX = position.x + curWidth / 2;
        let tooltipCenterY = position.y - 90;

        tooltipConfig.html = `
        <div class='tooltip-tb' style='display: flex; flex-direction: column;
          justify-content: center; font-weight: 600;'>
          <div> Layer ${d.layer + 1} Head ${d.head + 1} </div>
          <div style='font-size: 12px; opacity: 0.6;'> Semantic: ${round(d.semantic, 2)} </div>
          <div style='font-size: 12px; opacity: 0.6;'> Syntactic ${round(d.syntactic, 2)} </div>
          <div style='font-size: 12px; opacity: 0.6;'> Importance: ${round(d.confidence, 2)} </div>
        </div>
        `;
        tooltipConfig.width = 130;
        tooltipConfig.maxWidth = 130;
        tooltipConfig.left = tooltipCenterX - tooltipConfig.width / 2;
        tooltipConfig.top = tooltipCenterY;
        tooltipConfig.fontSize = '0.8em';
        tooltipConfig.show = true;
        tooltipConfigStore.set(tooltipConfig);

        // Show the background rect
        let curDonut = d3.select(e.currentTarget);

        if (!curDonut.classed('selected')){
          curDonut.select('.donut-rect')
            .style('opacity', 1);
        }
      })
      .on('mouseleave', (e) => {
        // TODO
        // let datum = d3.select(e.target).data()[0];
        // if (datum.layer === 2 && datum.head === 9) {
        //   return;
        // }

        tooltipConfig.show = false;
        tooltipConfigStore.set(tooltipConfig);

        // Hide the background rect
        let curDonut = d3.select(e.currentTarget);
        if (!curDonut.classed('selected')){
          curDonut.select('.donut-rect')
            .style('opacity', 0);
        }

      })
      .on('click', (e, d) => {
        sideInfo.show = true;
        sideInfo.attention = attentions[d.layer][d.head];
        sideInfo.tokens = saliencies.tokens.map(d => { return { 'token': d.token }; });
        sideInfo.layer = d.layer;
        sideInfo.head = d.head;
        sideStore.set(sideInfo);
      })
      .on('dblclick', (e) => {
        // console.log('double!');
        let curDonut = d3.select(e.currentTarget);
        if (curDonut.classed('selected')) {
          // pass
        } else {
          // Restore the currently selected rect
          let preDonut = d3.select(
            donutGroup.select(`#donut-rect-${curLayer}-${curHead}`)
              .node().parentNode
          );

          preDonut.select('.donut-rect')
            .style('fill', 'hsl(0, 0%, 80%)')
            .style('opacity', 0);
          
          preDonut.classed('selected', false);

          // Style the new rect
          curDonut.select('.donut-rect')
            .style('fill', 'hsl(27, 47%, 13%)')
            .style('opacity', 1);
          
          curDonut.classed('selected', true);

          curLayer = +curDonut.data()[0].layer;
          curHead = +curDonut.data()[0].head;

          mapHead.layer = curLayer;
          mapHead.head = curHead;
          mapHeadStore.set(mapHead);
        }
      });

    // Draw horizontal lines between rows
    donutGroup.selectAll('g.row-line-group')
      .data(Array(layerNum - 1).fill(0).map( (_, i) => i))
      .join('g')
      .attr('class', 'row-line-group')
      .append('path')
      .attr('d', d => {
        return `M${-maxOutRadius}
        ${(layerNum - d - 1 - 1/2) * (maxOutRadius * 2 + adjustedColGap)}
        L${headNum * (maxOutRadius * 2 + adjustedRowGap) - maxOutRadius}
        ${(layerNum - d - 1 - 1/2) * (maxOutRadius * 2 + adjustedColGap)}`;
      })
      .style('stroke', 'hsla(0, 0%, 0%, 0.1)');

    // Draw the label names
    let nameGroup = svg.append('g')
      .attr('class', 'name-group')
      .attr('transform', `translate(${SVGPadding.left}, ${maxOutRadius + headNameHeight})`);
    
    nameGroup.selectAll('g.layer-name-group')
      .data(Array(layerNum).fill(0).map( (_, i) => i))
      .join('g')
      .attr('class', 'layer-name-group')
      .attr('transform', d => `translate(${layerNameWidth - 10},
        ${(layerNum - d - 1) * (maxOutRadius * 2 + adjustedColGap)})`)
      .append('text')
      .text(d => d > 0 ? d + 1 : `Layer ${d + 1}`);

    let headNameGroup = svg.append('g')
      .attr('class', 'name-group')
      .attr('transform', `translate(${SVGPadding.left + layerNameWidth + maxOutRadius},
        ${9})`);

    headNameGroup.selectAll('g.head-name-group')
      .data(Array(layerNum).fill(0).map( (_, i) => i))
      .join('g')
      .attr('class', 'head-name-group')
      .attr('transform', d => `translate(${d * (maxOutRadius * 2 + adjustedRowGap)},
        ${0})`)
      .append('text')
      .text(d => d > 0 ? d + 1 : `Head ${d + 1}`);

    d3.select(viewContainer)
      .select('.head-arrow')
      .style('top', `${70 - 40}px`)
      .style('left', `${availableWidth - 170}px`);

    d3.select(viewContainer)
      .select('.layer-arrow')
      .style('top', `${70}px`)
      .style('left', `${availableWidth + 10}px`);

    let curDonut = d3.select(
      donutGroup.select(`#donut-rect-${curLayer}-${curHead}`)
        .node().parentNode
    );

    // Style the new rect
    curDonut.select('.donut-rect')
      .style('fill', 'hsl(27, 47%, 13%)')
      .style('opacity', 1);
    
    curDonut.classed('selected', true);
  };

  const drawDonut = (d, i, g, scales) => {
    let donut = d3.select(g[i]);

    let outRadius = scales.outRadiusScale(d.confidence);
    let ringRadius = scales.ringRadiusScale(d.confidence);
    let inRadius = Math.max(0, outRadius - ringRadius);

    // Draw the background rect
    let maxLength = 2 * scales.outRadiusScale.range()[1];
    donut.append('rect')
      .attr('class', 'donut-rect')
      .attr('id', `donut-rect-${d.layer}-${d.head}`)
      .attr('x', - maxLength / 2)
      .attr('y', - maxLength / 2)
      .attr('rx', 5)
      .attr('width', maxLength)
      .attr('height', maxLength)
      .style('fill', 'hsl(0, 0%, 80%)')
      .style('opacity', 0);

    // Draw an invisible circle for interaction
    donut.append('circle')
      .attr('cx', 0)
      .attr('cy', 0)
      .attr('r', outRadius)
      .style('fill', '#FDFCFC')
      .style('opacity', 1);

    // Draw the rings
    // Arc's center is at (0, 0) on the local coordinate
    let arc = d3.arc()
      .outerRadius(outRadius)
      .innerRadius(inRadius)
      .startAngle(0)
      .endAngle(Math.PI * 2);

    let color = d3.hcl(scales.hueScale(d.syntactic - d.semantic));
    color.l = scales.lightnessScale(Math.max(d.semantic, d.syntactic));

    donut.append('path')
      .attr('class', 'donut-chart')
      .attr('d', arc)
      .style('fill', color);

    // Draw the edges
    // Figure out the token positions
    let tokenPos = [];
    for (let i = 0; i < tokenSize; i++) {
      let curAngle = -Math.PI / 2 + i * (Math.PI * 2 / tokenSize);
      tokenPos.push({
        x: Math.cos(curAngle) * inRadius,
        y: Math.sin(curAngle) * inRadius,
        token: d.token,
        id: i
      });
    }

    // Create the links
    let links = [];

    // Todo
    // let threshold = 0.2;
    let threshold = 0;

    for (let i = 0; i < tokenSize; i++) {
      for (let j = 0; j < tokenSize; j++) {
        let curAttention = attentions[d.layer][d.head][i][j];
        if (curAttention > threshold) {
          links.push({
            source: i,
            target: j,
            attention: curAttention,
            id: `${i}-${j}`
          });
        }
      }
    }

    links = links.sort((a, b) => b.attention - a.attention).slice(0, 150);

    // Define link width scale
    let linkWidthScale = d3.scaleLinear()
      .domain(d3.extent(links.map(d => d.attention)))
      .range([0.2, 0.7]);

    let linkOpacityScale = d3.scaleLinear()
      // .domain(d3.extent(links.map(d => d.attention)))
      .domain([0, 1])
      .range([0.1, 1]);

    // Draw the links as bezier curves
    donut.selectAll('path.donut-link')
      .data(links, d => d.id)
      .join('path')
      .attr('class', 'donut-link')
      .attr('d', d => {
        let source = tokenPos[d.source];
        let target = tokenPos[d.target];
        const center = {x: 0, y: 0};
        const radialCurveAlpha = 2 / 5;
        
        // Two control points symmetric regarding the center point
        let controlP1 = {
          x: center.x + (source.x - center.x) * radialCurveAlpha,
          y: center.y + (source.y - center.x) * radialCurveAlpha
        };

        let controlP2 = {
          x: center.x + (target.x - center.x) * radialCurveAlpha,
          y: center.y + (target.y - center.x) * radialCurveAlpha
        };
        
        return `M ${source.x},${source.y} C${controlP1.x}, ${controlP1.y},
          ${controlP2.x}, ${controlP2.y}, ${target.x},${target.y}`;
      })
      .style('fill', 'none')
      .style('stroke', color)
      .style('pointer-events', 'none')
      .style('stroke-width', d => linkWidthScale(d.attention))
      .style('opacity', d => linkOpacityScale(d.attention));

  };

  const initData = async (attentionFile, saliencyFile, atlasFile) => {
    // Init attention data
    attentions = await d3.json(attentionFile);

    // init atlas data
    atlasData = await d3.json(atlasFile);
    
    // Init saliency data
    saliencies = await d3.json(saliencyFile);
    saliencies = saliencies[instanceID];
    tokenSize = saliencies.tokens.length;
  };

  const closeClicked = () => {
    sideInfo.show = false;
    sideStore.set(sideInfo);
    dispatch('close');
  };

  const badgeClicked = () => {
    if (isShown) {
      dispatch('close');
      isShown = false;

      d3.select(viewContainer)
        .select('.svg-container')
        .transition('move')
        .duration(700)
        .ease(ease)
        .style('opacity', 0);

      // Change the badge style
      d3.timer(() => {
        let badge = d3.select(viewContainer)
          .select('.badge')
          .style('border-left', '1px solid hsl(0, 0%, 90.2%)')
          .style('border-radius', '5px')
          .style('box-shadow', '-3px 3px 3px hsla(0, 0%, 0%, 0.06)')
          .style('margin-left', '5px');
        
        badge.select('.badge-title')
          .style('visibility', 'hidden');

        badge.select('.icon-wrapper > img')
          .attr('src', 'PUBLIC_URL/figures/map-marked-alt-solid.svg');
      }, 400);
    } else {
      dispatch('open');
      isShown = true;

      d3.select(viewContainer)
        .select('.svg-container')
        .transition('move')
        .duration(700)
        .ease(ease)
        .style('opacity', 1);

      // Change the badge style
      d3.timer(() => {
        let badge = d3.select(viewContainer)
          .select('.badge')
          .style('border-left', null)
          .style('border-radius', '0 5px 5px 0')
          .style('box-shadow', null)
          .style('margin-left', null);
        
        badge.select('.badge-title')
          .style('visibility', 'visible');

        badge.select('.icon-wrapper > img')
          .attr('src', 'PUBLIC_URL/figures/chevron-right-solid.svg');
      }, 400);
    }
  };

  onMount(async () => {
    // Load the attention and atlas data
    if (attentions == null || atlasData == null || saliencies == null) {
      initData(
        attentionDataDir + `attention-${padZeroLeft(instanceID, 4)}.json`,
        saliencyDataFilepath,
        atlasDataFilepath
      );
    }
  });

  instanceIDStore.subscribe(async value => {
    if (value !== instanceID) {
      instanceID = value;
      saliencies = await d3.json(saliencyDataFilepath);
      saliencies = saliencies[instanceID];
      tokenSize = saliencies.tokens.length;
      svg.select('*').remove();
      createGraph();
    }
  });

  sideStore.subscribe(value => {sideInfo = value;});

  mapViewConfigStore.subscribe(async value => {
    if (value.compHeight !== undefined && value.compWidth !== undefined){
      if (instanceViewConfig === undefined ||
        (instanceViewConfig.compHeight !== value.compHeight &&
        instanceViewConfig.compWidth !== value.compWidth)
      ){
        instanceViewConfig = value;
        
        SVGWidth = instanceViewConfig.compWidth;
        SVGHeight = instanceViewConfig.compHeight;

        // Load the attention and atlas data
        if (attentions == null || atlasData == null || saliencies == null) {
          initData(
            attentionDataDir + `attention-${padZeroLeft(instanceID, 4)}.json`,
            saliencyDataFilepath,
            atlasDataFilepath
          ).then(createGraph);
        } else {
          createGraph();
        }
      }
    }
  });
  

</script>

<style type='text/scss'>

  @import 'define';

  .svg-container {
    width: 100%;
    height: 100%;
    overflow: hidden;
    position: relative;
    cursor: default;

    display: flex;
    flex-direction: row;
    justify-content: space-around;
    align-items: flex-end;
  }

  .atlas-view {
    display: flex;
    flex-direction: row;
    max-width: 100%;
    position: relative;
    height: 100%;
    overflow: hidden;
    transition: max-width 1000ms ease-in-out;

    border-radius: 10px 0 0 10px;
    border-bottom: 1px solid hsla(0, 0%, 0%, 0.1);
    box-shadow: -3px -3px 3px hsla(0, 0%, 0%, 0.1);
    background: change-color($color: $brown-icon, $lightness: 99%);

    .triangle {
      content: "";
      position: absolute;
      top: 500px;
      left: 0;
      border-top: 20px solid transparent;
      border-bottom: 20px solid transparent;
      border-left: 20px solid hsl(0, 0%, 95%);
    }
  }

  .legend-container {
    padding-bottom: 25px;
    padding-top: 50px;
    height: 100%;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    pointer-events: none;

    :first-child {
      margin-bottom: 35px;
    }
  }

  .bottom-images {
    display: flex;
    flex-direction: column;

    :first-child {
      margin-bottom: 35px;
    }
  }

  .atlas-svg-container {
    position: relative;

    svg {
      border-right: 1px solid hsl(0, 0%, 90%);
    }
  }

  .head-arrow {
    position: absolute;
    width: 160px;
    pointer-events: none;
  }

  .layer-arrow {
    position: absolute;
    height: 160px;
    pointer-events: none;
  }

  .control-row {
    position: absolute;
    top: 0;
    left: 0;
    cursor: default;
    padding-top: 5px;
    display: flex;
    flex-direction: row;
    align-items: center;
    user-select: none;
    font-size: 0.9rem;
    z-index: 5;
  }

  .lower-atlas-label {
    color: hsl(0, 0%, 50%);
    font-size: 1.3rem;
    margin: 0 20px 0 10px;
    display: flex;
    flex-direction: row;
  }

  .expand-button {
    padding: 3px 5px;
    font-size: 1em;
    display: flex;
    align-items: center;
    cursor: pointer;
  }

  .select-row {
    position: relative;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: flex-end;

    border-radius: 5px;
    border: 1px solid change-color($brown-dark, $lightness: 90%);
    margin-right: 10px;

    &:hover {
      background: change-color($brown-dark, $alpha: 0.05);
    }
  }

  .icon-wrapper {
    display: flex;
    flex-direction: row;
    align-items: center;
    opacity: 0.5;
    transform: rotate(45deg);

    img {
      height: 1.2em;
    }
  }

  .hidden {
    visibility: hidden;
  }

</style>

<div class='atlas-view' bind:this={viewContainer}>

  <div class='triangle' class:hidden={!sideInfo.show}></div>

  <div class='control-row'>

    <div class='lower-atlas-label'>
      <div class='select-row'>
        <div class='relation-container' on:click={closeClicked}>
          <div class='expand-button'>
            <div class='icon-wrapper'>
              <img src='PUBLIC_URL/figures/arrow-forward-outline.svg' alt='expanding icon'>
            </div>
          </div>
        </div>
      </div>

      Attention Head Overview
    </div>

  </div>

  <div class='svg-container'>

    <div class='atlas-svg-container'>
      <svg class='atlas-svg-full'></svg>
    </div>

    <div class='legend-container'>
      <div>
        <img src='PUBLIC_URL/figures/click.png' width='180px' alt='click guide'>
      </div>
      <div class='bottom-images'>
        <img src='PUBLIC_URL/figures/size-legend.png' width='160px' alt='size legend'>
        <img src='PUBLIC_URL/figures/legend.png' width='200px' alt='color legend'>
      </div>
    </div>

  </div>
  
</div>