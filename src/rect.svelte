<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";

  const treeData: Node[] = [];
  let nofNodesPerParent = 10;
  let el: any;

  const generateChart = () => {
    for (let i = 0; i < nofNodesPerParent; ++i) {
      treeData.push({
        name: (Math.random() * nofNodesPerParent).toString(),
        children: Array.from({ length: nofNodesPerParent }).map((x) => ({
          name: (Math.random() * nofNodesPerParent).toString(),
          children: Array.from({ length: nofNodesPerParent }).map((x) => ({
            name: "end",
            children: [],
          })),
        })),
      });
    }
  };

  const redraw = () => {
    for (let i = 0; i < nofNodesPerParent; ++i) {
      treeData.push({
        name: (Math.random() * nofNodesPerParent).toString(),
        children: Array.from({ length: nofNodesPerParent }).map((x) => ({
          name: (Math.random() * nofNodesPerParent).toString(),
          children: Array.from({ length: nofNodesPerParent }).map((x) => ({
            name: "end",
            children: [],
          })),
        })),
      });
    }

    var margin = { top: 20, right: 90, bottom: 30, left: 90 },
      width = 960 - margin.left - margin.right,
      height = 500 - margin.top - margin.bottom;
    var svg = d3
      // .select("d3-example")
      .create("svg")
      .attr("width", width + margin.right + margin.left)
      .attr("height", height + margin.top + margin.bottom)
      .attr("height", dx)
      .attr("viewBox", [-margin.left, -margin.top, width, 10])
      .attr(
        "style",
        "max-width: 100%; height: auto; font: 10px sans-serif; user-select: none;"
      )
      .append("g")
      .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

    var i = 0,
      duration = 750,
      root;
    var treemap = d3.tree().size([height, width]);
    root = d3.hierarchy({ name: "Base", children: treeData }, function (d) {
      return d.children;
    });
    root.x0 = height / 2;
    root.y0 = 0;
    root.children.forEach(collapse);

    update(root);

    return svg.node();

    function collapse(d) {
      if (d.children) {
        d._children = d.children;
        d._children.forEach(collapse);
        d.children = null;
      }
    }

    function update(source) {
      var treeData = treemap(root);
      var nodes = treeData.descendants(),
        links = treeData.descendants().slice(1);
      nodes.forEach(function (d) {
        d.y = d.depth * 180;
      });
      var node = svg.selectAll("g.node").data(nodes, function (d) {
        return d.id || (d.id = ++i);
      });
      var nodeEnter = node
        .enter()
        .append("g")
        .attr("class", "node")
        .attr("transform", function (d) {
          return "translate(" + source.y0 + "," + source.x0 + ")";
        })
        .on("click", click);
      nodeEnter
        .attr("class", "node")
        .attr("r", 1e-6)
        .style("fill", function (d) {
          return d.parent ? "rgb(39, 43, 77)" : "#fe6e9e";
        });
      nodeEnter
        .append("rect")
        .attr("rx", function (d) {
          if (d.parent) return d.children || d._children ? 0 : 6;
          return 10;
        })
        .attr("ry", function (d) {
          if (d.parent) return d.children || d._children ? 0 : 6;
          return 10;
        })
        .attr("stroke-width", function (d) {
          return d.parent ? 1 : 0;
        })
        .attr("stroke", function (d) {
          return d.children || d._children
            ? "rgb(3, 192, 220)"
            : "rgb(38, 222, 176)";
        })
        .attr("stroke-dasharray", function (d) {
          return d.children || d._children ? "0" : "2.2";
        })
        .attr("stroke-opacity", function (d) {
          return d.children || d._children ? "1" : "0.6";
        })
        .attr("x", 0)
        .attr("y", -10)
        .attr("width", function (d) {
          return d.parent ? 40 : 20;
        })
        .attr("height", 20);

      nodeEnter
        .append("text")
        .style("fill", function (d) {
          if (d.parent) {
            return d.children || d._children ? "#ffffff" : "rgb(38, 222, 176)";
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

      var nodeUpdate = nodeEnter.merge(node);

      nodeUpdate
        .transition()
        .duration(duration)
        .attr("transform", function (d) {
          return "translate(" + d.y + "," + d.x + ")";
        });
      var nodeExit = node
        .exit()
        .transition()
        .duration(duration)
        .attr("transform", function (d) {
          return "translate(" + source.y + "," + source.x + ")";
        })
        .remove();
      nodeExit.select("rect").style("opacity", 1e-6);
      nodeExit.select("rect").attr("stroke-opacity", 1e-6);
      nodeExit.select("text").style("fill-opacity", 1e-6);
      var link = svg.selectAll("path.link").data(links, function (d) {
        return d.id;
      });
      var linkEnter = link
        .enter()
        .insert("path", "g")
        .attr("class", "link")
        .attr("d", function (d) {
          var o = { x: source.x0, y: source.y0 };
          return diagonal(o, o);
        });
      var linkUpdate = linkEnter.merge(link);
      linkUpdate
        .transition()
        .duration(duration)
        .attr("d", function (d) {
          return diagonal(d, d.parent);
        });
      var linkExit = link
        .exit()
        .transition()
        .duration(duration)
        .attr("d", function (d) {
          var o = { x: source.x, y: source.y };
          return diagonal(o, o);
        })
        .remove();
      nodes.forEach(function (d) {
        d.x0 = d.x;
        d.y0 = d.y;
      });
      function diagonal(s, d) {
        let path = `M ${s.y} ${s.x}
            C ${(s.y + d.y) / 2} ${s.x},
              ${(s.y + d.y) / 2} ${d.x},
              ${d.y} ${d.x}`;

        return path;
      }
      function click(d) {
        if (d.children) {
          d._children = d.children;
          d.children = null;
        } else {
          d.children = d._children;
          d._children = null;
        }
        update(d);
      }
    }
  };

  onMount(() => {
    const svg = redraw();
    console.log({ svg });

    el.innerHTML = "";
    el.appendChild(svg);
  });
</script>

<main>
  <div class="d3-example" bind:this={el} id="d3-example"></div>
</main>

<style>
  .node {
    cursor: pointer;
  }
  .node circle {
    fill: #fff;
    stroke: steelblue;
    stroke-width: 1.5px;
  }
  .node text {
    font: 10px sans-serif;
  }
  .link {
    fill: none;
    stroke: #ccc;
    stroke-width: 1.5px;
  }

  body {
    overflow: hidden;
  }
</style>
