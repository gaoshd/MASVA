<script setup lang="ts">
    import { ref, onMounted } from 'vue'
    import { repeat } from 'seemly'
    import * as d3 from 'd3'

    const values = ref([54, 76, 49, 90])
    const options = repeat(50000, undefined).map((_, i) => ({
        label: 'trajs_episode_' + String(i),
        value: i
      }))
    
    onMounted(() => {
        drawRewardOverview()
        drawRelationForce()
        drawToolTip()
        drawReplayGif(values.value)
        drawActionLine(values.value)
    })

    function drawRewardOverview(){
        const w  = 1500, h = 300, m = 30;
        const width = w - m*2, height = h - m*2;
        const labelPadding = m-5, tickBleed = 5;
            // Viewport configuration
        const svg = d3.select("#rewardOverViewContainer")
                .append("svg")
                .attr("height",h)
                .attr("width",w);

        svg.append("defs")
            .append("clipPath").attr("id", "chart")
            .append("rect")
            .attr("width", width)
            .attr("height", height);

        const chart = svg.append("g")
            .attr("transform", `translate(${[m, m]})`);
        
        const view = chart.append("g")
            .attr("class", "view")
            .attr("clip-path", "url(#chart)")
            .append("g") // the clipped zoomable object

        const play_1_line = d3.line();
        const play_2_line = d3.line();
        const scaleY = d3.scaleLinear().range([height, 0]);
        const scaleX = d3.scaleLinear().range([0, width]);
        const axisX = d3.axisBottom(scaleX) // this scale doesn't implement ticks!
            .tickPadding(1)

        const axisY = d3.axisLeft(scaleY).ticks(10)
                .tickPadding(tickBleed)
                .tickSizeOuter(0);

        const axisZero = d3.axisRight(scaleY)
                .tickValues([15])
                .tickSizeOuter(0)
                .tickSizeInner(width);
        const xG = chart.append("g").attr("class", "x-axis")
            .attr("transform", `translate(${[0, height]})`);
        const yG = chart.append("g").attr("class", "y-axis");
        const zeroG = chart.append("g").attr("class", "zero-axis");

        xG.append("text")
                .attr("class","label")
                .text("episodes")
                .attr("transform", `translate(${[(width/2),labelPadding]})`)

        yG.append("text")
                .attr("class","label")
                .text("reward")
                .attr("transform", `translate(${[-labelPadding,(height/2)]}) rotate(90)`)
        const extent = [[0,0],[width, height]];
        const zoom = d3.zoom()
                .scaleExtent([1, 130])
                .extent(extent)
                .translateExtent(extent)
                .on('zoom', zoomed);

        svg.call(zoom);

        function zoomed() {
            const t = d3.event.transform;
            console.log(t)
            //console.log(t.x,t.y,t.k)
            // rescale scales and axes
            const newScaleX = t.rescaleX(scaleX)
            axisX.scale(newScaleX);
            xG.call(axisX);

            // rescale horizontal scale for line chart
            play_1_line.x((d,i) => newScaleX(i))
            play_2_line.x((d,i) => newScaleX(i))
            d3.select(".play_1_line").datum(data.values).attr("d", play_1_line);
            d3.select(".play_2_line").datum(data.values).attr("d", play_2_line);
        }

        const data = {};
        d3.json('../public/trajs_json/split_episode_2_epoch_data.json')
            .then(function(result: any) {
                data.values = result
                console.log(data.values.length)
                // Scales domain configuratoon
                scaleY.domain([0, 25])
                scaleX.domain([0, data.values.length -1])


                // Axis rendering
                xG.call(axisX);
                yG.call(axisY);
                // zeroG.call(axisZero).select("line").attr("stroke", "green").attr("stroke-width", "2");

                // Line chart
                play_1_line.x((d,i) => scaleX(i+1))
                    .y(d => scaleY(d.score[1]));

                view.append("path").attr("class", "play_1_line")
                    .datum(data.values)
                    .attr("d", play_1_line)
                    .style("stroke", "LimeGreen")
                    .style("fill", "none");
                
                play_2_line.x((d,i) => scaleX(i))
                    .y(d => scaleY(d.score[0]));

                view.append("path").attr("class", "play_2_line")
                    .datum(data.values)
                    .attr("d", play_2_line)
                    .style("stroke", "SandyBrown")
                    .style("fill", "none");
        });
    }

    function drawRelationForce() {
        var relationContainer = d3.select("#relationForceContainer");
        const width = 1000;
        const height = 600;
        const margin = 100;

        var scale = d3.scaleLinear()
            .domain([0, 10])  // 输入域
            .range([0, 1]);
            
        const rcolorScale = d3.interpolateHslLong("navy", "red")
        const nodes = [{node: -1}]
        d3.range(100).forEach(i => {
            nodes.push({node: i})
        });
        d3.json('./trajs_json/trajs_top_3_data.json').then(links_data => {
            const links = links_data['array']
            const sim = d3.forceSimulation(nodes);
            sim.force("center", d3.forceCenter());

            const offsetX = margin/2 + width/2;
            const offsetY = margin/2 + height/2;

            const relate_svg = relationContainer.append("svg").attr("width",width).attr("height",height);
            const chart = relate_svg.append("g")
                    .attr("transform", `translate(${[offsetX,offsetY]})`);

            sim.force("link", d3.forceLink(links).iterations(15));
            sim.force("manybody", d3.forceManyBody().strength(100))
            sim.force("collide", d3.forceCollide(40));

            sim.on("tick", redraw);

            const zoom = d3.zoom()
                .on('zoom', function() {
                    chart.attr("transform",
                        d3.event.transform.translate(offsetX, offsetY));
                });
            relate_svg.call(zoom);

            // Dragging behavior
            const nodeDragged = d3.drag()
                    .on('drag', function(d) {
                        d.x = d3.event.x;
                        d.y = d3.event.y;
                    })
                    .on('start', function() {
                        if(sim.alpha() <= sim.alphaMin()) {
                            sim.restart();
                        }
                        sim.alphaTarget(sim.alphaMin() + .1)
                    })
                    .on('end', () => sim.alphaTarget(0));

            draw();
            function draw() {
                // svg.on('click', addNode);

                chart.selectAll('line')
                        .data(links, d => d.source.node + '-' + d.target.node)
                        .join('line')
                        .attr("x1", d => d.source.x)
                        .attr("x2", d => d.target.x)
                        .attr("y1", d => d.source.y)
                        .attr("y2", d => d.target.y)
                        .attr("stroke", "black")
                        .style('stroke-width', d => d.value * d.value)

                chart.selectAll('g.node')
                        .data(nodes, d => d.node)
                        .join("g").attr("class", 'node')
                        .style("transform", d => `translate(${[d.x, d.y]})`)
                        .each(function(d,i) {
                            d3.select(this).append("circle")
                                .attr("r", d => {return getRelateNodeInfo(d).relate_number + 15})
                                .style("fill", d => {return rcolorScale(scale(getRelateNodeInfo(d).relate_number))})
                                .style("stroke", 'black')
                            d3.select(this).append("text")
                                .text(d => d.node).attr("text-anchor", "middle")  
                                .attr("dominant-baseline", "middle") 
                        }).on("mouseover", d => highlight(d, true)).on("mouseout", d => highlight(d, false)).call(nodeDragged);
            }

            function getRelateNodeInfo(d) {
                const relate_info = {}
                let relate_number = 0;
                var relate_node = [d.node]
                var line = relate_svg.selectAll("line")
                    .filter(function(d1, i1) {        
                        if(d1.source.node == d.node || d1.target.node == d.node) {
                            relate_number++
                            relate_node.push(d1.source.node)
                            relate_node.push(d1.target.node)
                            return true
                        }
                    });
                relate_info.relate_node = relate_node
                relate_info.relate_number = relate_number
                return relate_info
            }
        
            function highlight(d, ishigh){
                var relate_info = getRelateNodeInfo(d)
                var line = relate_svg.selectAll("line")
                    .filter(function(d1, i1) {   
                        if(d1.source.node == d.node || d1.target.node == d.node) {
                            return true
                        }
                    });

                var node = relate_svg.selectAll(".node")
                    .filter(function(d1, i1) {    
                        if(relate_info.relate_node.includes(d1.node)){
                            return true
                        } 
                    });
                
                if(ishigh){
                    line.style('stroke-width', 5).attr("stroke", "red")
                    node.select("circle").style("stroke", 'red').style('stroke-width', 3)
                }else {
                    line.style('stroke-width', 1).attr("stroke", "black")
                    node.select("circle").style("stroke", 'black').style('stroke-width', 1)
                }
            }

            function redraw() {
                chart.selectAll('g.node')
                        .attr("transform", d => `translate(${[d.x, d.y]})`);

                chart.selectAll('line')
                        .attr("x1", d => d.source.x)
                        .attr("x2", d => d.target.x)
                        .attr("y1", d => d.source.y)
                        .attr("y2", d => d.target.y)
            }
        });
        
        
        
    }

    function drawToolTip() {
        var tooltip = d3.select("body")
            .append("div")
            .attr("class", "tooltip_container")
            .style("position", "absolute")
            .style("background", "#fff")
            .style("padding", "5px")
            .style("border", "1px solid #000")
            .style("border-radius", "5px")
            .style("opacity", 1);
    }
    
    function handleUpdateValue (value) {
        drawReplayGif(value)
        drawActionLine(value)
    }

    function drawReplayGif(value) {
        var imageContainer = d3.select("#replayImgContainer");
        imageContainer.selectAll(".image-container").remove()
        value.forEach(i => {
            // replay-view
            var imageName = "output_fast_"+ i + ".gif";
            var container = imageContainer.append("div").attr("class", "image-container").style("text-align", "center");
            container.append("text").text(imageName).attr("fill", "blue").style("font-size", "20px");
            container.append("img").attr("src", "../public/gif/" + imageName);
        });
    }

    function drawActionLine(value) {
        var actionLineContainer = d3.select("#actionLineContainer");
        actionLineContainer.selectAll("svg").remove()

        d3.json('./trajs_json/actions_data.json').then(d => {
            const data = d['array']
            const width = 1500
            const height = 200
            const compare_trajs = value
            const actionSvg = actionLineContainer.append("svg").attr("width", 1500).attr("height", (compare_trajs.length + 1) * 100)

            var xScale = d3.scaleLinear()
                            .domain([-5, 200])
                            .range([0, width]);
        
            var colorScale = d3.scaleOrdinal()
                .domain([0, 1, 2, 3, 4, 5])  // 定义数据的离散取值范围
                .range(["#b3b3b3", "#b3b3b3", "#8bcc00", "#0066cc", "#8bcc00", "#0066cc"]); 
                
            compare_trajs.forEach((d1, i1) => {
                const actionData = data[d1]
                let path_y = []
                const tooltip = d3.select('.tooltip_container')
                const g = actionSvg.append("g").attr("class", `action_line_g_${i1}`).on("mouseover", function(d) {
                    const path_element = d3.select(this).select("path")
                    path_element.style("stroke-width", 5).style("stroke", "red")
                    tooltip.style("opacity", .9);
                    tooltip.html(`${d1}`)
                        .style("left", (d3.event.pageX + 10) + "px")
                        .style("top", (d3.event.pageY - 28) + "px");
                }).on("mouseout", function(d) {
                    const path_element = d3.select(this).select("path")
                    path_element.style("stroke-width", 2).style("stroke", "gray")
                    tooltip.style("opacity", 0);
                });
                var line = d3.line()
                    .x(function (d, i) { return xScale(i); })
                    .y(function (d, i) {
                        var y;
                        if (i === 0 ){
                            y = 30
                        } else {
                            if (d === 2) {
                                y = path_y[i - 1] - 6
                            } else if (d === 3) {
                                y = path_y[i - 1] + 6
                            } else if (d === 4) {
                                y = path_y[i - 1] - 3
                            } else if (d === 5) {
                                y = path_y[i - 1] + 3
                            } else {
                                y = path_y[i - 1]
                            }
                        }
                        path_y.push(y)
                        return y
                    }
                ); 
                g.append("path")
                .datum(actionData)
                .attr("fill", "none")
                .attr("stroke", "gray" )
                .attr("stroke-width", 2)
                .attr("transform", "translate(50, 80)")
                .attr("d", line)

                g.selectAll("circle")
                    .data(actionData)
                    .enter()
                    .append("circle")
                    .attr("transform", "translate(50, 80)")
                    .attr("cx", function (d, i) { return xScale(i); })
                    .attr("cy", function (d, i) { { return path_y[i]; } })
                    .attr("r", 2) // 圆点的半径
                    .attr("fill", function (d) { return colorScale(d); });
            })
            var xAxis = d3.axisBottom(xScale).ticks(50)
            actionSvg.append("g")
                .attr("transform", "translate(50," + (height - 20) + ")")
                .call(xAxis);
        })
    }


</script>

<template>
    <div id="rewardOverViewContainer"></div>
    <div id="relationForceContainer"></div>
    <n-space vertical size="large">
        <n-select class="select-input" placeholder="选择 Trajs 进行分析" v-model:value="values" multiple :options="options" @update:value="handleUpdateValue" />
    </n-space>
    <div id="replayImgContainer"></div>
    <div id="actionLineContainer"></div>
</template>

<style scoped>
    .label {
        fill: black;
        font-size: 12pt;
        text-anchor: middle;
        font-size: 8pt;
    }
    .zero-axis line {
        stroke-opacity: .3;
    }
    .zero-axis text {
        opacity: 0;
    }
    .select-input {
        margin: 20px;
    }
    #replayImgContainer{
        margin-left: 20px;
        display: grid; /* 设置父元素为网格容器 */
        grid-template-columns: repeat(6, 1fr); /* 定义四列，每列平分父容器的宽度 */
        gap: 10px; /* 设置列之间的间隔 */
    }
</style>
