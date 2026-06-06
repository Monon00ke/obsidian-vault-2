---
layout: default
title: "Graph View"
permalink: /graph.html
---

<script src="https://d3js.org/d3.v7.min.js"></script>
<style>
    .graph-container { width: 100%; height: 70vh; background: #fff; border-radius: 8px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); overflow: hidden; position: relative; }
    .node circle { cursor: pointer; }
    .node text { font-size: 10px; fill: #333; pointer-events: none; }
    .link { stroke: #999; stroke-opacity: 0.6; }
    .tooltip { position: absolute; background: #24292e; color: #fff; padding: 6px 12px; border-radius: 4px; font-size: 12px; pointer-events: none; opacity: 0; transition: opacity 0.2s; }
    .controls { position: absolute; top: 10px; right: 10px; display: flex; gap: 8px; z-index: 10; }
    .controls button { background: #fff; border: 1px solid #e1e4e8; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-size: 12px; }
    .controls button:hover { background: #f1f8ff; }
    .legend { position: absolute; bottom: 10px; left: 10px; background: rgba(255,255,255,0.9); padding: 10px; border-radius: 4px; font-size: 11px; }
    .legend-item { display: flex; align-items: center; gap: 6px; margin: 4px 0; }
    .legend-dot { width: 10px; height: 10px; border-radius: 50%; }
</style>

<header>
    <h1>🕸️ Graph View</h1>
    <p>Interactive map of all notes. Click nodes to open notes, drag to rearrange.</p>
</header>

<div class="graph-container" id="graph">
    <div class="controls">
        <button onclick="zoomIn()">+</button>
        <button onclick="zoomOut()">-</button>
        <button onclick="resetZoom()">Reset</button>
    </div>
    <div class="legend" id="legend"></div>
    <div class="tooltip" id="tooltip"></div>
</div>

<div style="margin-top: 1rem; text-align: center;">
    <a href="{{ '/' | relative_url }}" class="back-link">← Back to Home</a>
</div>

<script>
const folderColors = {
    'English': '#0366d6',
    'Functional Analysis': '#6f42c1',
    'Math': '#d73a49',
    'OS Labs': '#28a745',
    'Photos': '#e36209',
    'Scientific Work': '#586069'
};

function getFolder(path) {
    return path.split('/')[0];
}

function getNodeColor(node) {
    const folder = getFolder(node.path);
    return folderColors[folder] || '#999';
}

d3.json("{{ '/graph.json' | relative_url }}").then(data => {
    const container = document.getElementById('graph');
    const width = container.clientWidth;
    const height = container.clientHeight;

    const svg = d3.select('#graph')
        .append('svg')
        .attr('width', width)
        .attr('height', height);

    const g = svg.append('g');

    const zoom = d3.zoom()
        .scaleExtent([0.1, 4])
        .on('zoom', (event) => g.attr('transform', event.transform));

    svg.call(zoom);

    window.zoomIn = () => svg.transition().duration(300).call(zoom.scaleBy, 1.5);
    window.zoomOut = () => svg.transition().duration(300).call(zoom.scaleBy, 0.67);
    window.resetZoom = () => svg.transition().duration(300).call(zoom.transform, d3.zoomIdentity);

    const simulation = d3.forceSimulation(data.nodes)
        .force('link', d3.forceLink(data.edges).id(d => d.id).distance(100))
        .force('charge', d3.forceManyBody().strength(-300))
        .force('center', d3.forceCenter(width / 2, height / 2))
        .force('collision', d3.forceCollide().radius(30));

    const link = g.append('g')
        .selectAll('line')
        .data(data.edges)
        .join('line')
        .attr('class', 'link')
        .attr('stroke-width', 1.5);

    const node = g.append('g')
        .selectAll('g')
        .data(data.nodes)
        .join('g')
        .attr('class', 'node')
        .call(d3.drag()
            .on('start', dragStarted)
            .on('drag', dragged)
            .on('end', dragEnded));

    node.append('circle')
        .attr('r', 6)
        .attr('fill', d => getNodeColor(d))
        .attr('stroke', '#fff')
        .attr('stroke-width', 2);

    node.append('text')
        .attr('dx', 10)
        .attr('dy', 4)
        .text(d => d.id.length > 20 ? d.id.substring(0, 18) + '...' : d.id);

    const tooltip = document.getElementById('tooltip');

    node.on('mouseover', function(event, d) {
        tooltip.style.opacity = 1;
        tooltip.innerHTML = `<strong>${d.id}</strong><br>${d.path}`;
    })
    .on('mousemove', function(event) {
        tooltip.style.left = (event.pageX + 10) + 'px';
        tooltip.style.top = (event.pageY - 10) + 'px';
    })
    .on('mouseout', function() {
        tooltip.style.opacity = 0;
    })
    .on('click', function(event, d) {
        const url = d.path.replace('.md', '.html');
        window.open("{{ '/' | relative_url }}" + url, '_blank');
    });

    // Build legend
    const legend = document.getElementById('legend');
    Object.entries(folderColors).forEach(([folder, color]) => {
        const count = data.nodes.filter(n => getFolder(n.path) === folder).length;
        if (count > 0) {
            legend.innerHTML += `<div class="legend-item"><div class="legend-dot" style="background:${color}"></div>${folder} (${count})</div>`;
        }
    });

    simulation.on('tick', () => {
        link
            .attr('x1', d => d.source.x)
            .attr('y1', d => d.source.y)
            .attr('x2', d => d.target.x)
            .attr('y2', d => d.target.y);

        node.attr('transform', d => `translate(${d.x},${d.y})`);
    });

    function dragStarted(event, d) {
        if (!event.active) simulation.alphaTarget(0.3).restart();
        d.fx = d.x;
        d.fy = d.y;
    }

    function dragged(event, d) {
        d.fx = event.x;
        d.fy = event.y;
    }

    function dragEnded(event, d) {
        if (!event.active) simulation.alphaTarget(0);
        d.fx = null;
        d.fy = null;
    }
});
</script>
