<script context="module" lang="ts">
    export const prerender = true;
</script>

<script lang="ts">
    import { onMount } from "svelte";
    import * as d3 from 'd3';


    // global constants
    const tickMs = 1000;

    enum State {
        Working,
        Breaking,
        Paused
    }

    // settings
    let workRatio = 3; // work time/break time

    // state
    let timerState = State.Paused;

    let chartData = [{time: 0, prop: 1}];
    let workMs = 0;
    let curWorkMs = 0;

    let restMs = 0;
    $: accRestMs = (workMs / workRatio) - restMs;

    let el;
    let interval;

    // chart consts
    const chartDims = { width: 640, height: 400 };
    const margins = { left: 40, right: 30, bottom: 30, top: 20 };

    const yScale = d3.scaleLinear([0, 1], [chartDims.height - margins.bottom, margins.top]);
    const yAxis = d3.axisLeft(yScale).ticks(chartDims.height / 40);

    let xScale;
    let xAxis;
    let xAxisG;

    let svg;

    let area1;
    let area1G;
    // let area2;
    // let area2G;

    onMount(async () => {
        xScale = d3.scaleLinear([0, d3.max(d3.map(chartData, x => x.time))], [margins.left, chartDims.width - margins.right])
        xAxis = d3.axisBottom(xScale);

        area1 = d3.area()
            .curve(d3.curveLinear)
            .x(i => xScale(chartData[i].time))
            .y0(yScale(0))
            .y1(i => yScale(chartData[i].prop));

        // area2 = d3.area()
        //     .curve(d3.curveLinear)
        //     .x(i => xScale(chartData[i].time))
        //     .y0(i => yScale(chartData[i].prop))
        //     .y1(yScale(1));

        interval = setInterval(() => tick(), tickMs);

        svg = d3.select(el)
            .append("svg")
            .attr("width", chartDims.width)
            .attr("height", chartDims.height)
            .attr("viewBox", [0, 0, chartDims.width, chartDims.height])
            .attr("class", "mx-auto")
            .attr("style", "max-width: 100%; height: auto; height: intrinsic;");

        svg.append("g")
            .attr("transform", `translate(${margins.left},0)`)
            .call(yAxis)
            .call(g => g.select(".domain").remove())
            .call(g => g.attr("class", "text-slate-500"))
            .call(g => g.selectAll(".tick line").clone()
                .attr("x2", chartDims.width - margins.left - margins.right)
                .attr("class", "text-slate-400 stroke-slate-400")
                .attr("stroke-opacity", 0.1))
            .call(g => g.append("text")
                .attr("x", - margins.left)
                .attr("y", 10)
                .attr("fill", "currentColor")
                .attr("class", "text-slate-600")
                .attr("text-anchor", "start")
                .text("prop"));

        area1G = svg.append("path")
            .attr("class", "fill-slate-400")
            .attr("d", area1(d3.range(chartData.length)));

        // area2G = svg.append("path")
        //     .attr("class", "fill-slate-600")
        //     .attr("d", area2(d3.range(chartData.length)));

        svg.append("line")
            .attr("x1", margins.left)
            .attr("x2", chartDims.width - margins.right)
            .attr("y1", yScale(workRatio / (workRatio + 1)))
            .attr("y2", yScale(workRatio / (workRatio + 1)))
            .style("stroke", "lightgreen")
            .style("stroke-width", 1.5);

        xAxisG = svg.append("g")
            .attr("transform", `translate(0, ${chartDims.height - margins.bottom})`)
            .attr("class", "text-slate-500")
            .call(xAxis);
    });

    function tick() {
        if (timerState === State.Working) {
            workMs += tickMs;
            curWorkMs += tickMs;

            if (chartData.length < (60 * 60 * 1000) / tickMs) {
                chartData = [...chartData, { time: workMs + restMs, prop: workMs / (workMs + restMs) }];
            } else {
                chartData = [...chartData.slice(1), { time: workMs + restMs, prop: workMs / (workMs + restMs) }];
            }
        } else if (timerState === State.Paused) {
            curWorkMs = 0;
        } else {
            curWorkMs = 0;
            restMs += tickMs;

            if (chartData.length < (60 * 60 * 1000) / tickMs) {
                chartData = [...chartData, { time: workMs + restMs, prop: workMs / (workMs + restMs) }];
            } else {
                chartData = [...chartData.slice(1), { time: workMs + restMs, prop: workMs / (workMs + restMs) }];
            }
        }

        // Update chart
        area1G.attr("d", area1(d3.range(chartData.length)));
        if (timerState === State.Paused) {
            xScale.domain([chartData[0].time, chartData[chartData.length - 1].time]);
        } else {
            xScale.domain([chartData[0].time, chartData[chartData.length - 1].time + tickMs]);
        }
        xAxisG.transition().call(d3.axisBottom(xScale));
    }

    function fmtMs(ms) {
        let neg = false;
        if (ms < 0) {
            neg = true;
            ms = -ms;
        }

        let secs = Math.floor(ms / 1000);
        let hours = Math.floor(secs / 3600);
        secs %= 3600;
        let mins = Math.floor(secs / 60);
        secs = secs % 60;

        return `${neg ? '-' : ''}${hours}:${('0' + mins).slice(-2)}:${('0' + secs).slice(-2)}`
    }

    function setStateFn(state: State) {
        return () => {
            timerState = state;
        }
    }
</script>

<svelte:head>
    <title>rational breaks</title>
</svelte:head>

<style>
    .chart :global(div) {
        font: 10px sans-serif;
        background-color: steelblue;
        text-align: right;
        padding: 3px;
        margin: 1px;
        color: white;
    }
</style>

<section class="min-h-screen bg-gray-800">
    <div class="container mx-auto text-center">
        <h1 class="pt-6 text-3xl text-gray-300 font-bold">rational breaks</h1>

        <div class="mt-10 rounded mx-auto max-w-md bg-slate-700 p-4">
            <div class="rounded p-2 border {timerState == State.Working ? 'border-slate-500' : 'border-slate-700'}">
                <p class="text-slate-400">work period</p>
                <p class="mt-1 font-mono text-5xl text-slate-300">{fmtMs(curWorkMs)}</p>
                <p class="mt-1 font-mono text-slate-400">({fmtMs(workMs)} total)</p>
            </div>

            <div class="mt-4 rounded p-2 border {timerState == State.Breaking ? 'border-slate-500' : 'border-slate-700'}">
                {#if accRestMs >= 0}
                    <p class="text-slate-400">acc. break period</p>
                    <p class="mt-1 font-mono text-5xl text-slate-300">{fmtMs(accRestMs)}</p>
                    <p class="mt-1 font-mono text-slate-400">(on break for {fmtMs(restMs)})</p>
                {:else}
                    <p class="text-red-400">work deficit</p>
                    <p class="mt-1 font-mono text-5xl text-red-300">{fmtMs(accRestMs * workRatio)}</p>
                    <p class="mt-1 font-mono text-red-400">break deficit: {fmtMs(accRestMs)}</p>
                    <p class="mt-1 font-mono text-slate-400">(on break for {fmtMs(restMs)})</p>
                {/if}
            </div>

            <div class="mt-6 inline-flex shadow-sm" role="group">
                <button
                    type="button"
                    on:click="{setStateFn(State.Working)}"
                    class="py-2 px-4 text-sm font-medium text-gray-900 bg-white rounded-l-lg border border-gray-200 hover:bg-gray-100 hover:text-blue-700 focus:z-10 focus:ring-2 focus:ring-blue-700 focus:text-blue-700 dark:bg-gray-700 dark:border-gray-600 dark:text-white dark:hover:text-white dark:hover:bg-gray-600 dark:focus:ring-blue-500 dark:focus:text-white"
                >
                    Working
                </button>
                <button
                    type="button"
                    on:click="{setStateFn(State.Paused)}"
                    class="py-2 px-4 text-sm font-medium text-gray-900 bg-white border-t border-b border-gray-200 hover:bg-gray-100 hover:text-blue-700 focus:z-10 focus:ring-2 focus:ring-blue-700 focus:text-blue-700 dark:bg-gray-700 dark:border-gray-600 dark:text-white dark:hover:text-white dark:hover:bg-gray-600 dark:focus:ring-blue-500 dark:focus:text-white"
                >
                    Paused
                </button>
                <button
                    type="button"
                    on:click="{setStateFn(State.Breaking)}"
                    class="py-2 px-4 text-sm font-medium text-gray-900 bg-white rounded-r-md border border-gray-200 hover:bg-gray-100 hover:text-blue-700 focus:z-10 focus:ring-2 focus:ring-blue-700 focus:text-blue-700 dark:bg-gray-700 dark:border-gray-600 dark:text-white dark:hover:text-white dark:hover:bg-gray-600 dark:focus:ring-blue-500 dark:focus:text-white"
                >
                    Taking a break
                </button>
            </div>
        </div>

        <div bind:this={el} class="chart mt-10 mx-auto"></div>
    </div>
</section>
