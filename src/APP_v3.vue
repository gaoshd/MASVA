<script setup lang="ts">
    import { ref, onMounted } from 'vue'
    import { repeat } from 'seemly'
    import * as d3 from 'd3'
    import * as echarts from 'echarts';
    import { useMessage, SelectOption } from 'naive-ui'
    import { matrix } from 'echarts/core';
    import { ListFormat, walkUpBindingElementsAndPatterns } from 'typescript';
    import {dvj} from './dvj-all.js'
    import 'd3-selection-multi'


    const curren_main_agent = ref(0)
    const curren_episode_id = ref(0)
    const curren_step_id = ref(0)
    let intervalId = ref(0)
    const current_step_indicator = ref('ball_speed')
    const current_steps = ref([])
    const current_body_indicator = ref("cfrc")


    onMounted(() => {
        handleUpdateRewardValue([], {label: 'nstep', value: '9'})
        drawToolTip()
    })

    const agentColorScale = d3.scaleOrdinal()
        .domain([0, 1, 2]) // 定义颜色对应的类别
        .range(["SteelBlue", "Tomato", "SeaGreen"]); // 定义颜色范围
    

    function drawToolTip() {
        var tooltip = d3.select("body")
            .append("div")
            .attr("class", "tooltip_container")
            .style("position", "absolute")
            .style("background", "#fff")
            .style("padding", "5px")
            .style("border", "1px solid #000")
            .style("border-radius", "5px")
            .style("opacity", 0.9)
            .attr("display", "none")
    }

    const overview_reward_options = [
        {label: 'total_reward', value: '0'}, 
        {label: 'nstep', value: '8'}, 
        {label: 'total_reward_forward', value: '1'},
        {label: 'total_reward_ctrl', value: '2'},
        {label: 'total_reward_contact', value: '3'},
        {label: 'total_reward_survive', value: '4'},
        {label: 'total_reward_goal_dist', value: '5'},
        {label: 'total_reward_move', value: '6'},
        {label: 'total_reward_goal', value: '7'},
    ]
    const edge_options =[
        {label: 'agent_0', value: '0'}, 
        {label: 'agent_0', value: '1'},
        {label: 'all_agent', value: '2'},
    ]

    function handleUpdateRewardValue(value, options) {
        let values_type = options.label
        updateRewardOverview(values_type)
    }

    function updateRewardOverview(values_type){
        d3.select("#overViewContainer").select("svg").remove()
        drawRewardOverview(values_type)
    }

    function drawRewardOverview(values_type){
        const w  = 1100, h = 310, m = 50;
        const width = w - m*2, height = h - m*2;
        const labelPadding = m-5, tickBleed = 5;
            // Viewport configuration
        const svg = d3.select("#overViewContainer")
                .append("svg")
                .attr("height",h)
                .attr("width",w);

        svg.append("defs")
            .append("clipPath").attr("id", "chart")
            .append("rect")
            .attr("width", width)
            .attr("height", height);

        const chart = svg.append("g")
            .attr("transform", `translate(${[m+15, m]})`);
        
        const view = chart.append("g")
            .attr("class", "view")
            .attr("clip-path", "url(#chart)")
            .append("g") // the clipped zoomable object
        const legend_g = svg.append("g").attr("class", "legend").attr("transform", `translate(${[2 * m, 20]})`);
        let show_circle = [true, true]
        d3.range(2).forEach((agent_id) => {
            let legend = legend_g.append("g").attr("class", `legend_${agent_id}`)
            legend.append("rect").attr("width", m + 20).attr("height", 20).attr("fill", agentColorScale(agent_id)).attr("x", agent_id * 80).attr("y", 15)
            let legend_text = agent_id == 0 ? 'Kicker' : 'Defender'
            legend.append("text").text(legend_text).attr("fill", "white").attr("x", agent_id * 80+5).attr("y", 30)
            legend.on('click', () => {
                let display_type = show_circle[agent_id] ? "none" : "block"
                view.selectAll(`circle.agent_${agent_id}`).style("display", display_type)
                show_circle[agent_id] = !show_circle[agent_id]
            })
        })

        const scaleY = d3.scaleLinear().range([height, 0]);
        const scaleStep = d3.scaleLinear().range([height, 0]);
        const scaleX = d3.scaleLinear().range([0, width - 10]);
        const axisX = d3.axisBottom(scaleX) // this scale doesn't implement ticks!
            .tickPadding(1)

        const axisY = d3.axisLeft(scaleY).ticks(10)
                .tickPadding(tickBleed)
                .tickSizeOuter(0);


        const xG = chart.append("g").attr("class", "x-axis")
            .attr("transform", `translate(${[0, height]})`);
        const yG = chart.append("g").attr("class", "y-axis");

        xG.append("text")
                .attr("class","label")
                .text(`episodes`)
                .attr("fill", "black")
                .style("font-size", "16px")
                .attr("transform", `translate(${[(width/2),labelPadding]})`)

        yG.append("text")
                .attr("class","label")
                .text("reward")
                .attr("fill", "black")
                .style("font-size", "16px")
                .attr("transform", `translate(${[-(labelPadding+10),(height/2)]}) rotate(90)`)

        const extent = [[0,0],[width, height]];
        const zoom = d3.zoom()
                .scaleExtent([1, 130])
                .extent(extent)
                .translateExtent(extent)
                .on('zoom', zoomed);

        svg.call(zoom);

        function zoomed() {
            const t = d3.event.transform;
            const newScaleX = t.rescaleX(scaleX)
            axisX.scale(newScaleX);
            xG.call(axisX);

            // rescale horizontal scale for line chart
            d3.range(2).forEach(agent_id => {
                view.selectAll(`circle.agent_${agent_id}`)
                        .attr("cx", (d, i) => newScaleX(i))
            });
        }

        const data = {};
        d3.json('../public/football_data_v2/episode_total_data.json')
            .then(function(result) {
                data.values = result
                let reward_list = []
                let episodes_len = data.values.length
                d3.range(2).forEach(agent_id => {
                    var agent_reward_list = data.values.map(function(obj) {
                        return obj[values_type][agent_id];
                    });
                    reward_list.push(agent_reward_list)
                })
                let max_reward = d3.max(d3.merge([reward_list[0], reward_list[1]])) + 100
                let min_reward = d3.min(d3.merge([reward_list[0], reward_list[1]])) - 100

                // Scales domain configuratoon
                scaleY.domain([min_reward, max_reward])
                scaleX.domain([-1, episodes_len])
                scaleStep.domain([d3.min(data.values, d=>d.nstep), d3.max(data.values, d=>d.nstep)])

                // Axis rendering
                xG.call(axisX);
                yG.call(axisY);


                const guide_line = view.append("line")
                    .attr("stroke", "white")
                    .attr("stroke-width", 1.5)
                    .attr("stroke-dasharray", "5,5")
                    .attr("x1", scaleX(0))
                    .attr("y1", scaleY(min_reward))
                    .attr("x2", scaleX(0))
                    .attr("y2", scaleY(max_reward))

                d3.range(2).forEach(agent_id => {
                    const mean_line = view.append("line")
                        .attr("stroke", agentColorScale(agent_id))
                        .attr("stroke-width", 1)
                        .attr("stroke-dasharray", `10,${agent_id + 1}`)
                        .attr("stroke-opacity", 0.8)
                        .attr("x1", scaleX(0))
                        .attr("y1", scaleY(d3.mean(reward_list[agent_id])))
                        .attr("x2", scaleX(episodes_len))
                        .attr("y2", scaleY(d3.mean(reward_list[agent_id])));
                        
                    view.selectAll(`circle.agent_${agent_id}`)
                        .data(data.values)
                        .enter()
                        .append("circle").attr("class", (d, i) => `agent_${agent_id} episode_${d.episode[0]}`)
                        .attr("cx", (d, i) => scaleX(i))
                        .attr("episode", (d, i) => i)
                        .attr("agent_id", agent_id)
                        .attr("cy", d => scaleY(d[values_type][agent_id]))
                        .attr("r", d => {
                            return  d.winner[0] == agent_id ? 2.5 : 2
                        }) 
                        .attr("fill", agentColorScale(agent_id))
                        .attr("fill-opacity", d => {
                            return  d.winner[0] == agent_id ? 0.9 : 0.5
                        })
                        .on("mouseover", handleMouseOver)
                        .on("mouseout", handleMouseOut)
                        .on("click", handleClick)
                });

                let reward_circle_node = view.select(`circle.agent_${curren_main_agent.value}.episode_${curren_episode_id.value}`)
                curren_step_id.value = 0
                reward_circle_node.dispatch("click");

                function handleClick() {
                    let element = d3.select(this)
                    let e_data = element.data()[0]
                    let episode_number = element.attr("episode")
                    curren_episode_id.value = episode_number
                    curren_step_id.value = 0
                    handleUpdateEpisodeValue(current_step_indicator.value)
                    handleUpdateEdgeView(episode_number)
                    d3.select("#replayImgContainer img").remove()
                    d3.select("#replayImgContainer svg").remove()
                    d3.select("#replayImgContainer").append("img").attr("src", "../public/football_data_v2/gif/" + curren_episode_id.value + '.gif').attr("width", 350).attr("height", 200)
                    let image_info_svg = d3.select("#replayImgContainer").append("svg").attr("width", 400).attr("height", 100)
                    overview_reward_options.forEach((option,i) => {
                        let reward_type = option.label
                        let reward_number_0;
                        let reward_number_1;
                        let reward_name;
                        if(i == 0){
                            reward_name = episode_number + "_reward" + ': '
                            reward_number_0 = e_data[reward_type][0].toFixed(0)
                            reward_number_1 = e_data[reward_type][1].toFixed(0)
                        }else if (i == 1) {
                            reward_name = 'nstep: '
                            reward_number_0 = e_data[reward_type][0]
                            reward_number_1 = e_data[reward_type][1]
                        } else {
                            reward_name =  reward_type.slice(13) + ': '
                            reward_number_0 = e_data[reward_type][0].toFixed(0)
                            reward_number_1 = e_data[reward_type][1].toFixed(0)
                        }
                        let text = image_info_svg.append("text")
                            .attr("x", () => {return i < 4 ? 1 : 200})
                            .attr("y", () => {return i < 4 ? (i+1) * 20: (i-3) * 20})

                        text.append("tspan")
                            .text(reward_name)
                            .style("fill", () => {return i==0 ? agentColorScale(e_data.winner[0]): 'black'});

                        text.append("tspan")
                            .text(reward_number_0 + ', ')
                            .style("fill", agentColorScale(0));

                        text.append("tspan")
                            .text(reward_number_1)
                            .style("fill", agentColorScale(1));
                    })
                }

                function handleMouseOver(){
                    let element = d3.select(this)
                    const tooltip = d3.select('div.tooltip_container')
                    element.attr("r", 4)
                    let e_data = element.data()[0]
                    const keys = d3.keys(e_data);
                    let tip_info = '';
                    overview_reward_options.forEach(option => {
                        let reward_type =  option.label;
                        let reward_0 = e_data[reward_type][0].toFixed(2)
                        let reward_1 = e_data[reward_type][1].toFixed(2)
                        tip_info += reward_type + ": [" + reward_0 + "," + reward_1 + ']<br/>'
                    });
                    guide_line.attr("x1", element.attr("cx"))
                              .attr("x2", element.attr("cx"))
                              .attr("stroke", function() {
                                return agentColorScale(e_data.winner)
                              })
                    tooltip.html(`${tip_info}`)
                        .style("left", (d3.event.pageX + 10) + "px")
                        .style("top", (d3.event.pageY - 28) + "px")
                        .style("display", "block");
                }
                function handleMouseOut(){
                    let element = d3.select(this)
                    const tooltip = d3.select('div.tooltip_container')
                    let agent_id = element.attr("agent_id")
                    let e_data = element.data()[0]
                    element.attr("r", () => {
                            return  e_data.winner == agent_id ? 2.5 : 2
                        })                     
                    tooltip.style("display", "none");
                }

        });
    }

    function handleUpdateEdgeValue(){
        // todo
    }

    function handleUpdateEdgeView(){
        drawRelationForce("#relationForceView_0", 0)
        drawRelationForce("#relationForceView_1", 1)
        drawRelationChordView()
    }
    
    function drawRelationForce(divName, agent_id) {
        d3.select(`${divName} svg`).remove()
        var relationContainer = d3.select(divName);
        const width = 500;
        const height = 400;
        const margin = 50;
            
        d3.json(`../public/football_data_v2/episode_relate_json/agent_${agent_id}_view/episode_${curren_episode_id.value}.json`).then(data => {
            const links = data['links']
            const nodes = [{node: -1, winner: 0}]
            data['nodes'].forEach(node => {
                nodes.push(node)
            })

            const sim = d3.forceSimulation(nodes);

            sim.force("center", d3.forceCenter());
            sim.force("link", d3.forceLink(links).id(d => d.node).iterations(10));
            sim.force("manybody", d3.forceManyBody().strength(1))
            sim.force("collide", d3.forceCollide(40));
            const offsetX = margin/2 + width/2;
            const offsetY = margin/2 + height/2;

            const relate_svg = relationContainer.append("svg").attr("width",width).attr("height",height);
            const chart = relate_svg.append("g")
                    .attr("transform", `translate(${[offsetX,offsetY]})`);
            const chart_text = relate_svg.append("text").text(`agent_${agent_id}_view`).attr("fill", "black").attr("x", 10).attr("y", 20).style("")

            sim.on("tick", redraw);

            const zoom = d3.zoom()
                .on('zoom', function() {
                    chart.attr("transform",
                        d3.event.transform.translate(offsetX, offsetY));
                });
            relate_svg.call(zoom);

            chart.selectAll('line')
                    .data(links, d => d.source.node + '-' + d.target.node)
                    .join('line')
                    .attr("x1", d => d.source.x)
                    .attr("x2", d => d.target.x)
                    .attr("y1", d => d.source.y)
                    .attr("y2", d => d.target.y)
                    .attr("stroke", "black")
                    .style('stroke-width', d => d.value * d.value )

            chart.selectAll('g.node')
                    .data(nodes, d => d.node)
                    .join("g").attr("class", 'node')
                    .style("transform", d => `translate(${[d.x, d.y]})`)
                    .each(function(d,i) {
                        d3.select(this).append("circle")
                            .attr("r", d => {return getRelateNodeInfo(d).relate_number + 13})
                            .style("fill", d => {return agentColorScale(d.winner)})
                            .attr("fill-opacity", 0.8)
                            .style("opacity", d => {return d.node == -1 ? 0 : 1})
                            .style("stroke", 'white')
                        d3.select(this).append("text")
                            .attr("fill", "white")
                            .text(d => d.node).attr("text-anchor", "middle")  
                            .attr("dominant-baseline", "middle") 
                    }).on("mouseover", d => highlight(d, true)).on("mouseout", d => highlight(d, false)).on("click", d => {
                        let overview_svg = d3.select("#overViewContainer svg")
                        let agent_id = d.winner == 2 ? 0:d.winner
                        let reward_circle_node = overview_svg.select(`circle.agent_${agent_id}.episode_${d.node}`)
                        curren_step_id.value = 0
                        reward_circle_node.dispatch("click");
                        d3.select("#relationChordView svg").remove()
                    });

            function getRelateNodeInfo(d) {
                const relate_info = {}
                let relate_number = 0;
                var relate_node = [d.node]
                var line = relate_svg.selectAll("line")
                    .filter(function(d1, i1) {
                        if(d1.source.node == d.node){
                            relate_node.push(d1.target.node)
                            relate_number++
                            return true
                        }
                        if(d1.target.node == d.node){
                            relate_node.push(d1.source.node)
                            relate_number++
                            return true
                        }
                        return false
                    });
                relate_info.relate_node = relate_node
                relate_info.relate_number = relate_number
                return relate_info
            }
        
            function highlight(d, ishigh){
                var relate_info = getRelateNodeInfo(d)
                let overview_svg = d3.select("#overViewContainer svg")
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
                let reward_circle_nodes = overview_svg.selectAll("circle")
                        .filter((d1, i1) => {
                            if(relate_info.relate_node.includes(d1.episode)){
                                return true
                            }
                            return false
                        })
                
                if(ishigh){
                    reward_circle_nodes.attr("stroke", "black").attr("stroke-width", 2)
                    line.style('stroke-width', 3).attr("stroke", "seagreen")
                    node.select("circle").style("stroke", 'seagreen').style('stroke-width', 3)
                }else {
                    reward_circle_nodes.attr("stroke-width", 0)
                    line.style('stroke-width', 1).attr("stroke", "black")
                    node.select("circle").style("stroke", 'white').style('stroke-width', 1)
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

    function drawRelationChordView() {
        const width = 500;
        const height = 400;
        const margin = 30;
        d3.select("#relationChordView svg").remove()
        d3.json(`./football_data_v2/episode_relate_json/agent_0_view/episode_${curren_episode_id.value}.json`).then(relate_data => {
            const episodes = relate_data.nodes.map(d => d.node);
            const episodes_winner = relate_data.nodes.map(d => d.winner)
            const edges = relate_data.links

            const matrix = dvj.linksToMatrix(episodes, edges, true);
            const chord = d3.chord()
                    .padAngle(.2)

            const chords = chord(matrix);


            const radius = height/2 - 60;

            const ribbon = d3.ribbon().radius(radius);

            const svg = d3.select("#relationChordView").append("svg").attr("width",width).attr("height",height);
            const svg_text = svg.append("text").text(`relation_view`).attr("fill", "black").attr("x", 10).attr("y", 20).style("")
            const chart = svg.append("g").attr("transform", `translate(${[width/2+margin/4, height/2+margin/4]})`);
            chart.selectAll('path.ribbon')
                    .data(chords)
                    .enter().append("path").attr("class",'ribbon')
                    .attr("d", ribbon)
                    .style("opacity", .5)
                    .style("fill", d => "seagreen")
                    .attr("fill-opacity", 0.8)
                    .on("mouseover", highlightRibbon)
                    .on("mouseout", d => {
                        d3.selectAll("path").classed('faded', false);
                        d3.select('.tooltip').transition().style("opacity", 0);
                    });

            const arc = d3.arc().innerRadius(radius+2).outerRadius(radius+30);
            chart.selectAll('path.arc')
                    .data(chords.groups)
                    .enter().append("path").attr("class",'arc')
                    .attr("d", arc)
                    .style("fill", d => agentColorScale(episodes_winner[d.index]))
                    .attr("fill-opacity", 0.8)
                    .on("mouseover", highlightNode)
                    .on("mouseout", d => chart.selectAll("path").classed('faded', false));


            chart.selectAll("text")
                    .data(chords.groups)
                    .enter().append("text")
                    .attr("x", d => arc.centroid(d)[0])
                    .attr("y", d => arc.centroid(d)[1])
                    .text(d => episodes[d.index])
                    .style("fill", 'black')
                    .attr("transform",d => `rotate(${(arc.endAngle()(d) + arc.startAngle()(d))*90/Math.PI},${arc.centroid(d)})`);

            const tooltip = chart.append("g")
                                .attr("class", 'tooltip hidden')
                                .attr("transform", `translate(${[-75, -50]})`)
                                .style("opacity", 0)
            tooltip.append("rect")
                    .attr("width",150)
                    .attr("height",100)
                    .attr("rx", 10)
                    .attr("ry", 10)
                    .style("fill", 'white')
                    .style("opacity", .8)
                    .style("stroke",'seagreen');

            const textFrom = tooltip.append("text").attr('id', 'from')
                .attr("x", 20)
                .attr("y", 25)
                .text('From: ').each(function(d) {
                        d3.select(this).append('tspan').text('')
                        d3.select(this).append('tspan').attr("x",30).attr('dy', 15).text('');
                    });

            const textTo = tooltip.append("text").attr('id', 'to')
                    .attr("x", 20)
                    .attr("y", 75)
                    .text('To: ').each(function(d) {
                        d3.select(this).append('tspan').text('')
                        d3.select(this).append('tspan').attr("x",30).attr('dy', 15).text('');
                    });

            function highlightNode(node) {
                d3.selectAll("path.arc").classed('faded', d => !(d.index === node.index));
                d3.selectAll("path.ribbon").classed('faded', edge => !(edge.source.index === node.index));
            }
            function highlightRibbon(edge) {
                d3.selectAll("path.arc").classed('faded', node => !(node.index === edge.source.index || node.index === edge.target.index))
                d3.selectAll("path.ribbon").classed('faded', d => !(d === edge))
                d3.select('.tooltip').transition().style("opacity", 1);
                d3.select('#from tspan:nth-child(1)').text(episodes[edge.source.index]);
                d3.select('#to tspan:nth-child(1)').text(episodes[edge.target.index]);
                d3.select('#from tspan:nth-child(2)').text(edge.source.value);
                d3.select('#to tspan:nth-child(2)').text(edge.target.value);
            }

            function contrast(color) {
                const c = d3.rgb(color);
                return (c.r * 0.299 + c.g * 0.587 + c.b * 0.114) > 130 ? 'black' : 'white';
            }
        })
        
    }

    const episode_detail_options = [
        {
          label: 'infos',
          key: 'infos',
          children: [
                {label: "reward_forward", key: "reward_forward"},
                {label: "reward_ctrl", key: "reward_ctrl"},
                {label: "reward_contact", key: "reward_contact"},
                {label: "reward_survive", key: "reward_survive"},
                {label: "reward_goal_dist", key: "reward_goal_dist"},
                {label: "reward_move", key: "reward_move"},
                {label: "reward_remaining",key: "reward_remaining"},
                {label: "reward_goal_type",key: "reward_goal_type"},
                {label: "reward_goal",key: "reward_goal"},
                {label: "reward_ctrl",key: "reward_ctrl"},
            ]
        },
        {
          label: 'obs',
          key: 'obs',
          children: [
                {label: "ball_goal_x_dist",key: "ball_goal_x_dist"},
                {label: "ball_goal_y1_dist",key: "ball_goal_y1_dist"},
                {label: "ball_goal_y2_dist",key: "ball_goal_y2_dist"},
                {label: "ball_speed",key: "ball_speed"},
            ]
        },
    ]
    
    function handleUpdateEpisodeValue(value_names){
        current_step_indicator.value = value_names
        d3.selectAll(".episodeViewContent svg").remove()
        drawEpisodeView(value_names)
    }

    function drawEpisodeView(value_names){
        let w = 1500, h = 400, m = 50
        const width = w - 2*m
        d3.json(`./football_data_v2/episode_detail_json/episode_${curren_episode_id.value}.json`).then((result)=> {
            let episode_data = result
            let step_len = result[0].step_data.length

            let step_data = episode_data[curren_main_agent.value].step_data
            let all_actions_data = d3.merge(step_data.map(d => d.actions)).sort(d3.ascending);
            let actions_extent = d3.extent(all_actions_data)
            let all_body_inicator_data = step_data.map(d => {
                let indicators = []
                Object.entries(d.body_info).forEach((d1) => {
                    indicators.push(d1[1][current_body_indicator.value])
                })
                return indicators
            })
            let body_indicators_extent = d3.extent(d3.merge(all_body_inicator_data).sort(d3.ascending))

            let agent_infos_data  = []
            let agent_obs_data= []
            
            d3.range(2).forEach((agent_id) => {
                let infos_data = result[agent_id].step_data.map((d,i) => {
                    d.infos['step'] = i
                    return  d.infos
                })
                let obs_data = result[agent_id].step_data.map((d,i) => {
                    d.obs['step'] = i
                    return  d.obs
                })
                agent_infos_data.push(infos_data)
                agent_obs_data.push(obs_data)
            })
            let all_infos_data = d3.merge(agent_infos_data)

            let main_agent_data
            let all_indicator_values
            if (value_names.includes("reward")){
                main_agent_data = agent_infos_data[curren_main_agent.value]
                all_indicator_values = main_agent_data.map(d => d[value_names])

            } else {
                main_agent_data = agent_obs_data[curren_main_agent.value]
                all_indicator_values = d3.merge(main_agent_data.map(d => d[value_names]))
            }

            drawStepLineView()
            function drawStepLineView() {
                const step_line_height = 50
                const stepScale = d3.scaleLinear().range([0, width - 1.5 * m]).domain([0, step_len])
                const svg = d3.select("#stepLineView").append('svg')
                    .attr('height', step_line_height)
                    .attr('width', width)
                    .attr('font-family', 'sans-serif')
                    .attr('font-size', 12)
                
                const axisStep = d3.axisBottom(stepScale)

                const stepSelectG = svg.append("g").attr("class", "step_axis").attr("transform", `translate(${[m, 15]})`).call(axisStep)
                stepSelectG.append("text").text("steps").attr("transform", `translate(${[-30, 0]})`).attr("fill", "black").style("font-size", 15)
                var colorScale = d3.scaleLinear()
                    .domain(d3.extent(all_indicator_values))  // 输入范围
                    .range(["white", agentColorScale(curren_main_agent.value)]);

                const step_circle = stepSelectG.selectAll("circle")
                    .data(main_agent_data)
                    .enter()
                    .append("circle")
                    .attr("r", 3)
                    .attr("step", d => d.step)
                    .attr("cx", (d, i) => stepScale(i))
                    .attr("cy", -8)
                    .attr("stroke", "black")
                    .attr("class", d => `step_${d.step}`)
                    .attr("stroke-width", 0.2)
                    .attr("fill", d => {
                        return colorScale(d[value_names])
                    })
                    .on("mouseover", step_circle_event)
                    .on("mouseout", step_circle_event)
                    .on("click", updateOtherView)
                
                // init data
                const startValue = curren_step_id.value - 9 < 0 ? 0 : curren_step_id.value - 9;
                const endStepsValue = (curren_step_id.value + 10) >= step_len ? step_len: curren_step_id.value + 10
                current_steps.value = d3.range(startValue, endStepsValue)
                drawEpisodeValueImage()
                drawParallerLine()
                drawActionRidgelineView()
                actionPictogramView()


                function updateOtherView() {
                    curren_step_id.value = d3.select(this).data()[0].step
                    drawEpisodeValueImage()
                    const startValue = curren_step_id.value - 9 < 0 ? 0 : curren_step_id.value - 9;
                    const endStepsValue = (curren_step_id.value + 10) >= step_len ? step_len: curren_step_id.value + 10
                    current_steps.value = d3.range(startValue, endStepsValue)
                    drawParallerLine()
                    drawActionRidgelineView()
                    updatePictogramView()
                }

                function step_circle_event(){
                    const tooltip = d3.select('div.tooltip_container')
                    let current_step_circle = d3.select(this)
                    let step = current_step_circle.attr("step")
                    if(d3.event.type == "mouseout" ? false:true){
                        tooltip.style("opacity", .9);
                        tooltip.html(`${step}`)
                            .style("left", (d3.event.pageX + 10) + "px")
                            .style("top", (d3.event.pageY - 28) + "px")
                            .style("display", "block");
                        current_step_circle.attr("stroke", "black")
                        current_step_circle.attr("stroke-width", 2)
                        current_step_circle.attr("r", 4)
                    } else {
                        tooltip.style("display", "none");
                        current_step_circle.attr("stroke-width", 0.2)
                        current_step_circle.attr("r", 3)
                    }
                }
            }

            function updatePictogramView() {
                const joint_color_scale = d3.scaleLinear()
                    .domain([actions_extent[0], d3.median(actions_extent), actions_extent[1]])  // 输入范围
                    .range(["seagreen", "white", agentColorScale(curren_main_agent.value)]);

                const body_color_scale = d3.scaleLinear()
                    .domain([ d3.mean(body_indicators_extent), body_indicators_extent[1]])
                    .range(["white", agentColorScale(curren_main_agent.value)]);
                const joint_data = step_data[curren_step_id.value].joint_info
                const sort_joint_data = Object.entries(joint_data).sort((a, b) => b[1] - a[1])
                const ap_svg =  d3.select("#actionPictogramView svg")
                const body_g = ap_svg.select("g")
                sort_joint_data.forEach(([joint_name, joint_value]) => {
                    body_g.select(`circle.${joint_name}`).attr("fill", joint_color_scale(joint_value)).attr("value", joint_value)
                })
                const body_data =  step_data[curren_step_id.value].body_info
                const sort_body_data = Object.entries(body_data).map((d) => {
                    return {"body_name": d[0], "value": d[1][current_body_indicator.value]}
                }).sort((a, b) => b.value - a.value)
                sort_body_data.forEach(d => {
                    body_g.select(`rect.${d.body_name}`).attr("fill", body_color_scale(d.value)).attr("value", d.value)
                })
            }

            function actionPictogramView() {
                d3.select("#actionPictogramView svg").remove()
                const ap_width = 400, ap_height = 320, ap_margin = 20;
                const ap_svg = d3.select("#actionPictogramView")
                                    .append("svg")
                                    .attr("width", ap_width)
                                    .attr("height", ap_height)

                const styles_data = {
                    fill: "none",
                    stroke: "black",
                    "stroke-width": 2,
                    "stroke-opacity": 1
                };
                
                const circleData = {
                    cx: d => d.x,
                    cy: d => d.y,
                    r: d => d.r
                };
                const rectData = {
                    x: d => d.x,
                    y: d => d.y,
                    width: d => d.w,
                    height:  d => d.h,
                    rx: 10,
                    ry: 10,
                    transform: d => `rotate(${d.rotate}, ${d.x + d.w/2}, ${d.y + d.h/2})`
                };
                const head_pos = {x: ap_width/2 - 50, y: ap_margin + 20}
                const joint_r = 10
                const draw_joint_data = [
                    {name: 'root', x: head_pos.x, y: head_pos.y, r: 20},
                    {name: 'right_shoulder', x: head_pos.x - 60, y: head_pos.y + 40, r: joint_r},
                    {name: 'left_shoulder', x: head_pos.x + 60, y: head_pos.y + 40, r: joint_r},
                    {name: 'right_hip', x: head_pos.x - 50, y: head_pos.y + 130, r: joint_r},
                    {name: 'left_hip', x: head_pos.x + 50, y: head_pos.y + 130, r: joint_r},
                    {name: 'abdomen', x: head_pos.x, y: head_pos.y + 100, r: 15},
                    {name: 'right_knee', x: head_pos.x - 50, y: head_pos.y + 200, r: joint_r},
                    {name: 'left_knee', x: head_pos.x + 50, y: head_pos.y + 200, r: joint_r},
                    {name: 'right_elbow', x: head_pos.x - 105, y: head_pos.y + 105, r: joint_r},
                    {name: 'left_elbow', x: head_pos.x + 105, y: head_pos.y + 105, r: joint_r},
                ];
                const body_h = 20
                const draw_body_data = [
                    {name: 'torso', x: head_pos.x - 50, y: head_pos.y + 30, w: 100, h: body_h, rotate: 0},
                    {name: 'lwaist', x: head_pos.x - 40, y: head_pos.y + 60, w: 80, h: body_h, rotate: 0},
                    {name: 'pelvis', x: head_pos.x - 40, y: head_pos.y + 120,  w: 80, h: body_h, rotate: 0},
                    {name: 'right_upper_arm', x: head_pos.x - 115, y: head_pos.y + 60,  w: 60, h: body_h, rotate: 120},
                    {name: 'left_upper_arm', x: head_pos.x + 55, y: head_pos.y + 60,  w: 60, h: body_h, rotate: -120},
                    {name: 'right_lower_arm', x: head_pos.x - 140, y: head_pos.y + 135,  w: 60, h: body_h, rotate: 90},
                    {name: 'left_lower_arm', x: head_pos.x + 80, y: head_pos.y + 135,  w: 60, h: body_h, rotate: 90},
                    {name: 'right_thigh', x: head_pos.x - 75, y: head_pos.y + 155,  w: 50, h: body_h, rotate: 90},
                    {name: 'left_thigh', x: head_pos.x + 25, y: head_pos.y + 155,  w: 50, h: body_h, rotate: 90},
                    {name: 'right_shin', x: head_pos.x - 70, y: head_pos.y + 220,  w: 40, h: body_h, rotate: 90},
                    {name: 'left_shin', x: head_pos.x + 30, y: head_pos.y + 220,  w: 40, h: body_h, rotate: 90},
                    {name: 'right_foot', x: head_pos.x - 60, y: head_pos.y + 250, w: 20, h: body_h, rotate: 0},
                    {name: 'left_foot', x: head_pos.x + 40, y: head_pos.y + 250, w: 20, h: body_h, rotate: 0}
                ];
                const body_g = ap_svg.append("g").attr("class", "body_g")
                body_g.selectAll("circle")
                    .data(draw_joint_data).enter()
                    .append("circle")
                    .attr("class", d => d.name)
                    .attrs(styles_data)
                    .attrs(circleData)
                    .on("mouseover", bodyMouseEvent)
                    .on("mouseout", bodyMouseEvent)

                body_g.selectAll("rect")
                    .data(draw_body_data).enter()
                    .append("rect")
                    .attr("class", d => d.name)
                    .attrs(styles_data)
                    .attrs(rectData)
                    .on("mouseover", bodyMouseEvent)
                    .on("mouseout", bodyMouseEvent)

                function bodyMouseEvent() {
                    const tooltip = d3.select('div.tooltip_container')
                    const body_name = d3.select(this).attr("class")
                    const body_value = d3.select(this).attr("value")

                    const tipinfo = `${body_name}: ${body_value}`
                    if(d3.event.type == "mouseout" ? false:true){
                        tooltip.style("opacity", .9);
                        tooltip.html(`${tipinfo}`)
                            .style("left", (d3.event.pageX + 20) + "px")
                            .style("top", (d3.event.pageY + 10) + "px")
                            .style("display", "block")
                    } else {
                        tooltip.style("display", "none");
                    }
                }
                updatePictogramView()
            }

            function drawEpisodeValueImage(){
                d3.select("#obsView svg").remove()
                let v_i_width = 350, v_i_height = 400, v_i_margin = 10
                clearInterval(intervalId.value);
                let value_image_svg = d3.select("#obsView").append("svg").attr("width", v_i_width).attr("height", v_i_height)
                const step_len = episode_data[0].step_data.length
                let ball_data = episode_data[0].step_data
                let init_ball_pos = ball_data[curren_step_id.value].obs.ball_pos
                let init_ball_speed = ball_data[curren_step_id.value].obs.ball_speed
                let agent_0_data = episode_data[0].step_data
                let agent_1_data = episode_data[1].step_data
                let init_agent_0_pos = agent_0_data[curren_step_id.value].obs.agent_pos
                let init_agent_1_pos = agent_1_data[curren_step_id.value].obs.agent_pos
                let stop_play = true
                const xScale = d3.scaleLinear()
                    .domain([-5, 5]) 
                    .range([v_i_margin, v_i_width - v_i_margin]); 

                const yScale = d3.scaleLinear()
                    .domain([-6, 6]) 
                    .range([v_i_height - v_i_margin, v_i_margin]); 

                const xAxis = d3.axisBottom(xScale);
                const yAxis = d3.axisLeft(yScale);
                value_image_svg.append("g")
                            .attr("transform", `translate(0, ${v_i_height/2})`)
                            .call(xAxis);

                value_image_svg.append("g")
                            .attr("transform", `translate(${v_i_width/2}, 0)`)
                            .call(yAxis);
                value_image_svg.selectAll(".domain").remove()
                const obs_g = value_image_svg.append("g").attr("class", "obsG")
                obs_g.append("line").attr("x1", xScale(4)).attr("y1", yScale(-5)).attr("x2", xScale(4)).attr("y2", yScale(5)).attr("stroke", agentColorScale(1)).attr("stroke-width", 2)

                const obs_episode_text = obs_g.append("text").attr("transform", `translate(${v_i_margin}, ${v_i_margin})`).text(`episode: ${curren_episode_id.value}`)
                const obs_step_text = obs_g.append("text").attr("transform", `translate(${v_i_margin}, ${v_i_margin + 20})`).text(`step: ${curren_step_id.value}`)
                const obs_ball_speed_text = obs_g.append("text").attr("transform", `translate(${v_i_margin}, ${v_i_margin + 40})`).text(`ball_speed: ${init_ball_speed}`)


                const replay_button_g = obs_g.append("g").attr("transform", `translate(${v_i_margin}, ${v_i_margin + 60})`)
                replay_button_g.append("rect").attr("width", 50).attr("height", 20).attr("opacity", 0.5).attr("fill", "seagreen")
                replay_button_g.append("text").attr("x", 25).attr("y", 15).text("replay").attr("fill", "white").attr("text-anchor", "middle")
                replay_button_g.on("click", () => {
                    curren_step_id.value = 0
                    stop_play = true
                    intervalId.value = setInterval(updateValueImage, 50)
                    curren_step_id.value = 0
                })

                const next_button_g = obs_g.append("g").attr("transform", `translate(${v_i_margin}, ${v_i_margin + 85})`)
                next_button_g.append("rect").attr("width", 50).attr("height", 20).attr("opacity", 0.5).attr("fill", "seagreen")
                next_button_g.append("text").attr("x", 25).attr("y", 15).text("next").attr("fill", "white").attr("text-anchor", "middle")
                next_button_g.on("click", () => {
                    if (curren_step_id.value +1 == step_len){
                        curren_step_id.value = 0
                        return
                    }
                    updateValueImage()
                })


                const back_button_g = obs_g.append("g").attr("transform", `translate(${v_i_margin}, ${v_i_margin + 135})`)
                back_button_g.append("rect").attr("width", 50).attr("height", 20).attr("opacity", 0.5).attr("fill", "seagreen")
                back_button_g.append("text").attr("x", 25).attr("y", 15).text("back").attr("fill", "white").attr("text-anchor", "middle")
                back_button_g.on("click", () => {
                    console.log(curren_step_id.value)
                    if (curren_step_id.value -1 == -1){
                        curren_step_id.value = 0
                        return
                    }
                    updateValueImage(true)
                })

                const stop_button_g = obs_g.append("g").attr("transform", `translate(${v_i_margin}, ${v_i_margin + 110})`)
                stop_button_g.append("rect").attr("width", 50).attr("height", 20).attr("opacity", 0.5).attr("fill", "seagreen")
                stop_button_g.append("text").attr("x", 25).attr("y", 15).text("stop").attr("fill", "white").attr("text-anchor", "middle")
                stop_button_g.on("click", () => {
                    if(stop_play){
                        clearInterval(intervalId.value);
                        stop_play = false
                    } else {
                        intervalId.value = setInterval(updateValueImage, 50)
                        stop_play = true
                    }
                })
                
                const agents_g = value_image_svg.append("g").attr("class", "agentsG")

                const agent_0_ele = agents_g.append("circle").attr("class", "kicker").attr("agent", 0).attr("r", 5).attr("cx", xScale(init_agent_0_pos[0])).attr("cy", yScale(init_agent_0_pos[1])).attr("fill", agentColorScale(0)).on("mouseover", agent_event).on("mouseout", agent_event)

                const agent_1_ele = agents_g.append("circle").attr("class", "defender").attr("r", 5).attr("agent", 1).attr("cx", xScale(init_agent_1_pos[0])).attr("cy", yScale(init_agent_1_pos[1])).attr("fill", agentColorScale(1)).on("mouseover", agent_event).on("mouseout", agent_event)

                const ball_ele = agents_g.append("circle").attr("class", "ball").attr("r", 5).attr("cx", xScale(init_ball_pos[0])).attr("cy", yScale(init_ball_pos[1])).attr("fill", "black").on("mouseover", ball_event).on("mouseout", ball_event)


                function agent_event() {
                    const tooltip = d3.select('div.tooltip_container')
                    let agent_name =  d3.select(this).attr('class')
                    let agent_id = d3.select(this).attr('agent')
                    let agent_data = agent_id == 0 ? agent_0_data[curren_step_id.value].obs:agent_1_data[curren_step_id.value].obs
                    console.log(agent_data)
                    const tipinfo = `${agent_name}_pos_z: ${agent_data.agent_pos[2]}<br/>`
                    if(d3.event.type == "mouseout" ? false:true){
                        tooltip.style("opacity", .9);
                        tooltip.html(`${tipinfo}`)
                            .style("left", (d3.event.pageX + 10) + "px")
                            .style("top", (d3.event.pageY - 28) + "px")
                            .style("display", "block")
                    } else {
                        tooltip.style("display", "none");
                    }
                }

                function ball_event() {
                    const tooltip = d3.select('div.tooltip_container')
                    let ball_data = agent_0_data[curren_step_id.value].obs
                    const tipinfo = `ball_qvel_x: ${ball_data.ball_qvel[0]}<br/>
                    ball_qvel_y: ${ball_data.ball_qvel[1]}<br/>
                    ball_qvel_z: ${ball_data.ball_qvel[2]}<br/>`
                    if(d3.event.type == "mouseout" ? false:true){
                        tooltip.style("opacity", .9);
                        tooltip.html(`${tipinfo}`)
                            .style("left", (d3.event.pageX + 10) + "px")
                            .style("top", (d3.event.pageY - 28) + "px")
                            .style("display", "block")
                    } else {
                        tooltip.style("display", "none")
                    }
                }
                
                function updateValueImage(is_back=false) {
                    console.log("is_back", is_back)
                    if (curren_step_id.value +1 == step_len){
                        return
                    }
                    curren_step_id.value = is_back ? curren_step_id.value - 1 : curren_step_id.value + 1 
                    const startValue = curren_step_id.value - 9 < 0 ? 0 : curren_step_id.value - 9;
                    const endStepsValue = (curren_step_id.value + 10) >= step_len ? step_len: curren_step_id.value + 10
                    current_steps.value = d3.range(startValue, endStepsValue)
                    drawParallerLine()
                    updatePictogramView()
                    if(curren_step_id.value > step_len - 1){
                        clearInterval(intervalId.value);
                        return
                    }
                    let ball_pos = ball_data[curren_step_id.value].obs.ball_pos
                    let ball_speed = ball_data[curren_step_id.value].obs.ball_speed
                    let agent_0_pos = agent_0_data[curren_step_id.value].obs.agent_pos
                    let agent_1_pos = agent_1_data[curren_step_id.value].obs.agent_pos
                    ball_ele.attr("cx", xScale(ball_pos[0])).attr("cy", yScale(ball_pos[1]))
                    agent_0_ele.attr("cx", xScale(agent_0_pos[0])).attr("cy", yScale(agent_0_pos[1]))
                    agent_1_ele.attr("cx", xScale(agent_1_pos[0])).attr("cy", yScale(agent_1_pos[1]))
                    obs_step_text.text(`step: ${curren_step_id.value}`)
                    obs_ball_speed_text.text(`ball_speed: ${ball_speed}`)
                }          
            }

            function drawParallerLine() {
                d3.select("#rewardParallerView svg").remove()
                const r_width = 1100, r_height = 400, r_margin = 50
                const reward_svg = d3.select("#rewardParallerView").append('svg')
                        .attr('height', r_height)
                        .attr('width', r_width)
                        .attr('font-family', 'sans-serif')
                        .attr('font-size', 12)

                const keys = ["step", "reward_goal_dist", "reward_contact", "reward_forward", "reward_survive","reward_ctrl", "reward_move", "reward_goal_type", "reward_remaining","reward_goal"]

                const y = new Map(Array.from(keys, key => [key, d3.scaleLinear(d3.extent(all_infos_data, d => Number(d[key])), [r_height - (r_margin + 30), 0])]))
                const x = d3.scalePoint(keys, [0, r_width - 2 * r_margin])
                const step_line = d3.line()
                    .defined(([, value]) => value != null)
                    .y(([key, value]) => y.get(key)(value))
                    .x(([key]) => x(key))
                    .curve(d3.curveCatmullRom)

                reward_svg.append('g')
                    .attr("transform", `translate(${[r_margin, r_margin -20]})`)
                    .selectAll('g')
                    .data(keys)
                    .join('g')
                    .attr("y", 300)
                    .attr('transform', d => `translate(${x(d)}, 0)`)
                    .each(function(d) { 
                        d3.select(this).call(d3.axisRight(y.get(d)))
                    })
                    .call(g => g.append('text')
                        .attr('y', -15)
                        .attr('x', -10)
                        .attr('text-anchor', 'start')
                        .attr('fill', 'currentColor')
                        .text(d => d))

                d3.range(2).forEach((agent_id) => {
                    let step_data = agent_infos_data[agent_id]
                    reward_svg.append('g')
                        .attr("transform", `translate(${[r_margin, r_margin -20]})`)
                        .attr("class", "step_line")
                        .attr('fill', 'none')
                        .attr('stroke-width', 2.4)
                        .attr('stroke-opacity', 0.5)
                        .attr('stroke-dasharray', '1 2.4')
                        .attr('stroke-linejoin', 'round')
                        .attr('stroke-linecap', 'round')
                        .selectAll('path')
                        .data(step_data.filter(ele => {
                            return current_steps.value.includes(ele.step)
                        }))
                        .join('path')
                        .attr("class", `reward_path agent_${agent_id}_reward_path`)
                        .attr('stroke', agentColorScale(agent_id))
                        .attr('d', d => step_line(d3.cross(keys, [d], (key, d) => [key, d[key]])))
                        .on("mouseover", rewardPathEvent).on("mouseout", rewardPathEvent)
                        .on("click", updateEpisodeViewByRewardPath)
                })
                function updateEpisodeViewByRewardPath() {
                    let path_data = d3.select(this).data()[0]
                    let stepline_svg = d3.select("#stepLineView svg")
                    let step_circle_node = stepline_svg.select(`circle.step_${path_data.step}`)
                    curren_step_id.value = path_data.step
                    step_circle_node.dispatch("click");
                }

                function rewardPathEvent() {
                    const tooltip = d3.select('div.tooltip_container')
                    let all_reward_path = d3.selectAll("#rewardParallerView .reward_path")
                    let reward_path = d3.select(this)
                    let path_data = d3.select(this).data()[0]
                    if(d3.event.type == "mouseout" ? false:true){
                        tooltip.style("opacity", .9);
                        tooltip.html(`step: ${path_data.step}`)
                            .style("left", (d3.event.pageX + 10) + "px")
                            .style("top", (d3.event.pageY - 28) + "px")
                            .style("display", "block")
                        all_reward_path.attr("stroke-opacity", 0.1)
                        reward_path.attr('stroke-opacity', 1).attr('stroke-width', 3)
                    }else {
                        tooltip.style("display", "none");
                        all_reward_path.attr("stroke-opacity", 0.5).attr('stroke-width', 2.4)
                    }
                    
                }
            }

            function drawActionRidgelineView(){
                d3.select("#actionRidgelineView svg").remove()
                const flatData = step_data.map(d => d.actions);;
                const action_names = ['abdomen', 'right_hip', 'right_knee', 'left_hip', 'left_knee', 'right_shoulder', 'right_elbow', 'left_shoulder', 'left_elbow']

                const series = d3.transpose(flatData).map((arr,i) => {
                    return {"name": action_names[i], "values": arr}
                }).sort((a, b) => d3.mean(b.values) - d3.mean(a.values));
                const h_series = series.map((d)=> {
                    return {"name": d.name, "values": d.values.slice(d3.extent(current_steps.value)[0], d3.extent(current_steps.value)[1] + 1)}
                })


                // Specify the chart’s dimensions.
                const overlap = 3;
                const a_width = 1100;
                const a_height = series.length * 35;
                const marginTop = 50;
                const marginBottom = 50;
                const marginLeft = 80;
                // Create the scales.
                const step_xScale = d3.scaleLinear()
                    .domain([0, step_len - 1])
                    .range([marginLeft, a_width]);

                const actions_yScale = d3.scalePoint()
                    .domain(series.map(d => d.name))
                    .range([marginTop, a_height - marginBottom]);

                const actions_zScale = d3.scaleLinear()
                    .domain(actions_extent).nice()
                    .range([0, -overlap * actions_yScale.step()]);

                const area = d3.area()
                    .curve(d3.curveBasis)
                    .defined(d => !isNaN(d))
                    .x((d, i) => step_xScale(d3.range(step_len)[i]))
                    .y0(0)
                    .y1(d => actions_zScale(d));

                const h_area = d3.area()
                    .curve(d3.curveBasis)
                    .defined(d => !isNaN(d))
                    .x((d, i) => {
                        return step_xScale(current_steps.value[i])})
                    .y0(0)
                    .y1(d => actions_zScale(d));

                const line = area.lineY1();
                const h_line = h_area.lineY1();

                // Create the SVG container.
                const actions_svg = d3.select("#actionRidgelineView").append("svg")
                    .attr("width", a_width)
                    .attr("height", a_height)
                    .attr("viewBox", [0, 0, a_width, a_height])
                    .attr("style", "max-width: 100%; height: auto;");

                const action_axis = actions_svg.append("g")
                    .attr("transform", `translate(${marginLeft},0)`)
                    .call(d3.axisLeft(actions_yScale).tickSize(0).tickPadding(4))
                    .call(g => g.select(".domain").remove());

                // Append a layer for each series.
                const group = actions_svg.append("g")
                    .selectAll("g")
                    .data(series)
                    .join("g")
                    .attr("class", (d, i) => `action_${i}`)
                    .attr("transform", d => `translate(0,${actions_yScale(d.name) + 30})`)
                    .append("path")
                    .attr("fill", agentColorScale(curren_main_agent.value))
                    .attr("fill-opacity", 0.2)
                    .attr("stroke", "black")
                    .attr("stroke-width", 0)
                    .attr("d", d => area(d.values))
                    .on("mouseover", action_group_event)
                    .on("mouseout", action_group_event)
                    .append("path")
                    .attr("fill", "none")
                    .attr("d", d => line(d.values));

                function action_group_event(){
                    const tooltip = d3.select('div.tooltip_container')
                    let current_action_group = d3.select(this)
                    let body_name = current_action_group.data()[0].name
                    let mouse_step = step_xScale.invert(d3.mouse(this)[0]).toFixed(0)
                    if(d3.event.type == "mouseout" ? false:true){
                        tooltip.style("opacity", .9);
                        tooltip.html(`body_name: ${body_name}<br/>
                        step: ${mouse_step}`)
                            .style("left", (d3.event.pageX + 10) + "px")
                            .style("top", (d3.event.pageY + 20) + "px")
                            .style("display", "block")
                        current_action_group.attr("stroke-width", 2)
                    } else {
                        tooltip.style("display", "none");
                        current_action_group.attr("stroke-width", 0)
                    }
                }

                function h_action_group_event(){
                    const tooltip = d3.select('div.tooltip_container')
                    let current_actions_data = d3.select(this).selectAll(".action_path").data()
                    let actions_data = []
                    current_actions_data.forEach((d) => {
                        let mean_value = d3.mean(d.values)
                        actions_data.push({"name": d.name, "value": mean_value.toFixed(3)})
                    })
                    let sort_actions_data = actions_data.sort((a, b) => b.value - a.value)
                    let tipinfo = ''
                    sort_actions_data.forEach((d) => {
                        tipinfo += `${d.name}: ${d.value} <br/>`
                    })
                    if(d3.event.type == "mouseout" ? false:true){
                        tooltip.style("opacity", .9);
                        tooltip.html(`${tipinfo}`)
                            .style("left", (d3.event.pageX + 10) + "px")
                            .style("top", (d3.event.pageY - 28) + "px")
                            .style("display", "block")
                    } else {
                        tooltip.style("display", "none");
                    }
                }

                // const h_group = actions_svg.append("g").on("mouseover", h_action_group_event)
                //     .on("mouseout", h_action_group_event)
                //     .selectAll("g")
                //     .data(h_series)
                //     .join("g")
                //     .attr("class", (d, i) => `action_${i} action_path`)
                //     .attr("transform", d => `translate(0,${actions_yScale(d.name) + 30})`)
                //     .append("path")
                //     .attr("fill", (d) => {
                //         return agentColorScale(curren_main_agent.value)})
                //     .attr("opacity", 0.7)
                //     .attr("d", d => h_area(d.values))
                //     .append("path")
                //     .attr("fill", "none")
                //     .attr("stroke", "black")
                //     .attr("d", d => h_line(d.values));
            }

        })
    }

    function railStyle({focused,checked}) {
        const style = {};
        if (checked) {
          style.background = "Tomato";
        } else {
          style.background = "SteelBlue";
        }
        return style;
    }
    
    function handleUpdateAgentTypeValue(value){
        if(value){
            curren_main_agent.value = 1
        }else {
            curren_main_agent.value = 0
        }
        handleUpdateEpisodeValue(current_step_indicator.value)
    }

    const body_values_options =[
        {label: 'vel', value: '0'}, 
        {label: 'qfrc', value: '1'},
        {label: 'cfrc', value: '2'},
    ]

    function handleUpdateBodyValue(value_names, options) {
        current_body_indicator.value = options.label
        handleUpdateEpisodeValue(current_step_indicator.value)
    }
    
</script>

<template>
    <div class="overView">
        <div class="bar">
            <div class="barName">overView</div>
            <div style="float: left; margin-left: 10px; margin-top: 2px; margin-right: 5px;">data_type:</div>
            <n-space class="rewardSelectEle" vertical>
                <n-select
                    size="small"
                    default-value="8"
                    :options="overview_reward_options"
                    @update:value="handleUpdateRewardValue"
                />
            </n-space>
        </div>
        <div class="overViewContent">
            <div style="border-right:1px solid black;" id="overViewContainer"></div>
            <div style="margin-top: 10px; margin-left: 20px;" id="replayImgContainer">
            </div>
        </div>
    </div>
    <div class="edgeView">
        <div class="bar">            
            <div class="barName">edgeView</div>
            <n-space class="edgeSelectEle" vertical>
                <n-select
                    size="small"
                    default-value="0"
                    :options="edge_options"
                    @update:value="handleUpdateEdgeValue"
                />
            </n-space>
        </div>
        <div class="edgeViewContent">
            <div style="border-right:1px solid black; margin-left: 5px;" id="relationForceView_0"></div>
            <div style="margin-left: 1px;" id="relationChordView"></div>
            <div style="border-left: 1px solid black; margin-left: 1px;" id="relationForceView_1"></div>
        </div>
    </div>
    <div class="episodeView">
        <div class="bar">            
            <div class="barName">episodeView</div>
            <text style="margin-left: 10px; margin-top: 5px; margin-right: 10px; float: left">step_indicator:</text>
            <n-tree-select
                style="float: left;"
                class="episodSelectEle"
                size="small"
                max-tag-count="responsive"
                default-value="ball_speed"
                :options="episode_detail_options"
                @update:value="handleUpdateEpisodeValue"
            />
            <n-switch style=" float: left;margin-left: 10px; margin-top: 3px;" :rail-style="railStyle" @update:value="handleUpdateAgentTypeValue">
                <template #checked>
                    Defender
                </template>
                <template #unchecked>
                    Kicker
                </template>
            </n-switch>
            <text style="margin-left: 10px; margin-top: 5px; margin-right: 10px; float: left">body_indicator:</text>
            <n-select
                size="small"
                style="float: left; margin-top: 1px; width: 100px;"
                default-value="2"
                :options="body_values_options"
                @update:value="handleUpdateBodyValue"
            />
        </div>
        <div class="episodeViewContent">
            <div style="margin-left: 5px;" id="stepLineView"></div>
            <div class="rewardAndObsViewContent">
                <div style="" id="rewardParallerView"></div>
                <div style="margin-left: 10px;" id="obsView"></div>
            </div>
            <div class="actionViewContent">
                <div id="actionRidgelineView"></div>
                <div style="margin-left: 10px;" id="actionPictogramView"></div>
            </div>
        </div>
    </div>

</template>

<style scoped>
    .barName {
        float: left;
        margin-top: 2px;
        font-size: 15px;
        margin-left: 10px;
    }
    .episodSelectEle {
        margin-top: 1px;
        width: 200px;
    }
    .episodeView {
        margin-left:5px;
        margin-top: 5px;
        display: flex;
        flex-direction: column;
    }
    .edgeViewContent {
        display: flex;
        flex-direction: row;
        margin-top: 10px;
    }
    .episodeViewContent{
        display: flex;
        flex-direction: column;
        margin-top: 10px;
    }
    .rewardAndObsViewContent {
        display: flex;
        flex-direction: row;
        margin-left: 5px;
    }
    .actionViewContent{
        display: flex;
        flex-direction: row;
        margin-left: 5px;
    }
    .rewardSelectEle {
        margin-top: 1px;
        width: 200px;
        float: left;
    }
    .edgeSelectEle {
        margin-top: 1px;
        margin-left: 100px;
        width: 200px;
    }
    .overView {
        margin-left: 5px;
        display: flex;
        flex-direction: column;
    }
    .bar{
        background-color: #2E8B57;
        height: 30px;
        color: white;
    }
    .overViewContent {
        margin-top: 10px;
        margin-bottom: 10px;
        display: flex;
        flex-direction: row;
    }
    .edgeView {
        margin-left: 5px;
        display: flex;
        flex-direction: column;
    }

    .episodeViewBar {
        background-color: black;
        height: 30px;
        color: white;
    }
    .faded {
        opacity: .1;
    }

</style>
