<script lang="ts">
  import * as d3 from "d3";
  import { BrowserText } from "./utils/helper";
  import { onMount } from "svelte";

  interface Node {
    name: string;
    children: Node[];
  }

  let el: any;

  const treeData: Node[] = [];
  let nofNodesPerParent = 20;
  let nofLevels = 2;

  const generateChart = () => {
    // Generating samples value for the tree
    const generateLevels = (depth: number): Node[] => {
      // Base case: if depth is 0, return an empty array (no children)
      if (depth === 0) {
        return [];
      }

      // Recursive case: generate children for each node
      return Array.from({ length: nofNodesPerParent }).map((x, index) => ({
        name: `Node Level ${depth} - Slave ${index + 1}`,
        children: generateLevels(depth - 1), // Decrease depth with each level
      }));
    };

    for (let i = 0; i < nofNodesPerParent; ++i) {
      treeData.push({
        name: "Hussien Alrekabi & Hussien ALabady",
        children: generateLevels(nofLevels),
      });
    }

    // Specify the charts’ dimensions. The height is variable, depending on the layout.
    const width = 2000;
    const marginTop = 40;
    const marginRight = 10;
    const marginBottom = 40;
    const marginLeft = 40;

    // Rows are separated by dx pixels, columns by dy pixels. These names can be counter-intuitive
    // (dx is a height, and dy a width). This because the tree must be viewed with the root at the
    // “bottom”, in the data domain. The width of a column is based on the tree’s height.
    const root = d3.hierarchy({ name: "Base", children: treeData });
    const dx = 40;
    const dy = (width - marginRight - marginLeft) / (1 + root.height);

    // Define the tree layout and the shape for links.
    const tree = d3.tree().nodeSize([dx, dy]);
    const diagonal = d3
      .linkHorizontal()
      .x((d: any) => d.y)
      .y((d: any) => d.x);

    // Add zoom behavior to the SVG
    const zoom = d3
      .zoom()
      .scaleExtent([2, 4])
      .on("zoom", (event) => {
        svg.attr("transform", event.transform); // Apply the zoom transformation
      });

    // Create the SVG container, a layer for the links and a layer for the nodes.
    const svg = d3
      .create("svg")
      .attr("fill", "#F3F4F6")
      .attr("height", dx)
      .attr("viewBox", [-marginLeft, -marginTop, width, dx])
      .attr(
        "style",
        "max-width: 100%; height: auto; font: 10px sans-serif; user-select: none;"
      )
      .call(zoom as any);

    const gLink = svg
      .append("g")
      .attr("fill", "none")
      .attr("stroke", "#555")
      .attr("stroke-opacity", 0.4)
      .attr("stroke-width", 1.5);

    const gNode = svg
      .append("g")
      .attr("cursor", "pointer")
      .attr("pointer-events", "all");

    function update(event: any, source: any) {
      const duration = event?.altKey ? 2500 : 200; // hold the alt key to slow down the transition
      const nodes = root.descendants().reverse();
      const links = root.links();

      // Compute the new tree layout.
      tree(root as any);

      let left: any = root;
      let right: any = root;
      root.eachBefore((node: any) => {
        if (node.x < left.x) left = node;
        if (node.x > right.x) right = node;
      });

      const height = right.x - left.x + marginTop + marginBottom;

      const transition = svg
        .transition()
        .duration(duration)
        .attr("height", height)
        .attr("viewBox", [
          -marginLeft,
          left.x - marginTop,
          width,
          height,
        ] as any)
        .tween(
          "resize",
          window.ResizeObserver
            ? (null as any)
            : () => () => svg.dispatch("toggle")
        );

      // Update the nodes…
      const node = gNode.selectAll("g").data(nodes, (d: any) => d.id);

      // Enter any new nodes at the parent's previous position.
      const nodeEnter = node
        .enter()
        .append("g")
        .attr("class", "node")
        .attr("transform", (d) => `translate(${source.y0},${source.x0})`)
        .attr("fill-opacity", 0)
        .attr("stroke-opacity", 0)
        .on("click", (event: any, d: any) => {
          d.children = d.children ? null : d._children;
          update(event, d);
        });

      nodeEnter
        .append("rect")
        .attr("rx", function (d) {
          return 12;
        })
        .attr("ry", function (d) {
          return 0;
        })
        .attr("stroke-width", function (d) {
          return d.parent ? 1 : 0;
        })
        .attr("fill", "white")
        .attr("stroke", function (d) {
          return "#e5e7eb";
          // return d.children || d._children
          //   ? "rgb(3, 192, 220)"
          //   : "rgb(38, 222, 176)";
        })
        .attr("stroke-dasharray", function (d: any) {
          return d.children || d._children ? "0" : "2.2";
        })
        .attr("stroke-opacity", function (d: any) {
          return d.children || d._children ? "1" : "0.6";
        })
        .attr("x", (d) => {
          const width = BrowserText.getWidth(d.data.name, 12, "Arial");

          return -width / 2 - 8;
        })
        .attr("y", -15)
        .attr("rx", 100)
        .attr("width", function (d) {
          const width = BrowserText.getWidth(d.data.name, 12, "Arial");

          return width + 50;
        })
        .attr("height", 30);

      nodeEnter
        .append("text")
        .style("fill", function (d) {
          if (d.parent) {
            return "black";
          }
          return "rgb(39, 43, 77)";
        })
        .attr("dy", ".35em")
        .attr("x", function (d) {
          return d.parent ? 20 : 10;
        })
        .attr("text-anchor", function (d) {
          return "middle";
        })
        .text(function (d) {
          return d.data.name;
        });

      // Transition nodes to their new position.
      const nodeUpdate = node
        .merge(nodeEnter as any)
        .transition(transition as any)
        .attr("transform", (d) => `translate(${d.y},${d.x})`)
        .attr("fill-opacity", 1)
        .attr("stroke-opacity", 1);

      // Transition exiting nodes to the parent's new position.
      const nodeExit = node
        .exit()
        .transition(transition as any)
        .remove()
        .attr("transform", (d) => `translate(${source.y},${source.x})`)
        .attr("fill-opacity", 0)
        .attr("stroke-opacity", 0);

      // Update the links…
      const link = gLink.selectAll("path").data(links, (d: any) => d.target.id);

      // Enter any new links at the parent's previous position.
      const linkEnter = link
        .enter()
        .append("path")
        .attr("d", (d) => {
          const o = { x: source.x0, y: source.y0 };
          return diagonal({ source: o, target: o } as any);
        });

      // Transition links to their new position.
      link
        .merge(linkEnter as any)
        .transition(transition as any)
        .attr("d", diagonal as any);

      // Transition exiting nodes to the parent's new position.
      link
        .exit()
        .transition(transition as any)
        .remove()
        .attr("d", (d) => {
          const o = { x: source.x, y: source.y };
          return diagonal({ source: o, target: o } as any);
        });

      // Stash the old positions for transition.
      root.eachBefore((d: any) => {
        d.x0 = d.x;
        d.y0 = d.y;
      });
    }

    // Do the first update to the initial configuration of the tree — where a number of nodes
    // are open (arbitrarily selected as the root, plus nodes with 7 letters).
    (root as any).x0 = dy / 2;
    (root as any).y0 = 0;
    root.descendants().forEach((d: any, i: number) => {
      d.id = i;
      d._children = d.children;
      if (d.depth && d.data.name.length !== 7) d.children = null;
      d.y = d.depth * 180;
    });

    update(null, root);

    return svg.node();
  };

  onMount(() => {
    const svg = generateChart();
    el.appendChild(svg);
  });
</script>

<main>
  <input type="number" bind:value={nofNodesPerParent} name="" id="" />
  <input type="number" bind:value={nofLevels} name="" id="" />
  <button
    on:click={() => {
      el.innerHTML = "";
      el.appendChild(generateChart());
    }}>Generate</button
  >

  <div id="el" class="chart" bind:this={el}></div>
</main>

<style>
</style>
