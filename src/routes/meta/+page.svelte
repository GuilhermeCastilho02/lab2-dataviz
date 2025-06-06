<svelte:head>
    <title>My page</title>
</svelte:head>

<script>
    import * as d3 from "d3";
    import Scrolly from "svelte-scrolly";

    import { onMount } from "svelte";

    import {
        computePosition,
        autoPlacement,
        offset,
    } from '@floating-ui/dom';

    import Bar from '$lib/Bar.svelte';
    import FileLines from "$lib/FileLines.svelte";

    let data = [];
    let commits = [];
    let width = 1000, height = 600;
    let margin = {top: 10, right: 10, bottom: 30, left: 20};
    let usableArea = {
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left
    };
    usableArea.width = usableArea.right - usableArea.left;
    usableArea.height = usableArea.bottom - usableArea.top;

    let xAxis, yAxis;
    let yAxisGridlines;
    let cursor = {x: 0, y: 0};
    
    $: minDate = d3.min(commits.map(d => d.date));
    $: maxDate = d3.max(commits.map(d => d.date));
    $: maxDatePlusOne = new Date(maxDate);
    $: maxDatePlusOne.setDate(maxDatePlusOne.getDate() + 1);

    $: xScale = d3.scaleTime()
                .domain([minDate, maxDatePlusOne])
                .range([margin.left, width - margin.right])
                .nice();

    $: yScale = d3.scaleLinear()
                .domain([24, 0])
                .range([height - margin.bottom, margin.top]);

    $: {
        d3.select(xAxis).call(d3.axisBottom(xScale));
        d3.select(yAxis).call(d3.axisLeft(yScale).tickFormat(d => String(d % 24).padStart(2, "0") + ":00"));
    }

    $: {
        d3.select(yAxisGridlines).call(
            d3.axisLeft(yScale)
            .tickFormat("")
            .tickSize(-usableArea.width)
        );
    }

    $: rScale = d3.scaleLinear()
                .domain(d3.extent(commits.map(d=>d.totalLines)))
                .range([2, 30]);

    let hoveredIndex = -1;
    $: hoveredCommit = commits[hoveredIndex] ?? hoveredCommit ?? {};

    let commitTooltip;
    let tooltipPosition = {x: 0, y: 0};

    let clickedCommits = [];

    $: allTypes = Array.from(new Set(data.map(d => d.type)));
    $: selectedLines = (clickedCommits.length > 0 ? clickedCommits : filteredCommits).flatMap(d => d.lines);
    $: selectedCounts = d3.rollup(
        selectedLines,
        v => v.length,
        d => d.type
    );
    $: languageBreakdown = allTypes.map(type => [type, selectedCounts.get(type) || 0]);

    let commitProgress = 100;
    let filteredCommits;
    $: timeScale = d3.scaleTime().domain([minDate,maxDatePlusOne]).range([0,100]);
    $: commitMaxTime = timeScale.invert(commitProgress);
    $: filteredCommits = commits.filter(commits => (commits.datetime < commitMaxTime));

    let filteredData;
    $: filteredData = data.filter(commits => (commits.datetime < commitMaxTime));   

    async function dotInteraction (index, evt) {
        let hoveredDot = evt.target;
        if (evt.type === "mouseenter") {
            hoveredIndex = index;
            cursor = {x: evt.x, y: evt.y};
            tooltipPosition = await computePosition(hoveredDot, commitTooltip, {
                strategy: "fixed", // because we use position: fixed
                middleware: [
                    offset(5), // spacing from tooltip to dot
                    autoPlacement() // see https://floating-ui.com/docs/autoplacement
                ],
            });        }
        else if (evt.type === "mouseleave") {
            hoveredIndex = -1
        }
        else if (evt.type === "click") {
            let commit = commits[index]
            if (!clickedCommits.includes(commit)) {
                // Add the commit to the clickedCommits array
                clickedCommits = [...clickedCommits, commit];
            }
            else {
                    // Remove the commit from the array
                    clickedCommits = clickedCommits.filter(c => c !== commit);
            }
        }
    }

    onMount(async () => {
        data = await d3.csv("./loc.csv", row => ({
            ...row,
            line: Number(row.line), // or just +row.line
            depth: Number(row.depth),
            length: Number(row.length),
            date: new Date(row.date + "T00:00" + row.timezone),
            datetime: new Date(row.datetime)
    }));

    commits = d3.groups(data, d => d.commit).map(([commit, lines]) => {
    let first = lines[0];
    let {author, date, time, timezone, datetime} = first;
    let ret = {
        id: commit,
        url: "https://github.com/GuilhermeCastilho02/lab2-dataviz/commit/" + commit,
        author, date, time, timezone, datetime,
        hourFrac: datetime.getHours() + datetime.getMinutes() / 60,
        totalLines: lines.length
    };
    commits = d3.sort(commits, d => -d.totalLines);

    // Like ret.lines = lines
    // but non-enumerable so it doesn’t show up in JSON.stringify
    Object.defineProperty(ret, "lines", {
        value: lines,
        configurable: true,
        writable: true,
        enumerable: false,
    });

    return ret;
});
});
</script>

<h1>Meta</h1>

<dl class="info tooltip" hidden={hoveredIndex === -1} style="top: {cursor.y}px; left: {cursor.x}px" bind:this={commitTooltip}>
    <dt>Commit</dt>
    <dd><a href="{ hoveredCommit.url }" target="_blank">{ hoveredCommit.id }</a></dd>

    <dt>Date</dt>
    <dd>{ hoveredCommit.datetime?.toLocaleString("en", {dateStyle: "full"}) }</dd>

    <dt>Author</dt>
    <dd>{ hoveredCommit.author }</dd>

    <dt>Time</dt>
    <dd>{ hoveredCommit.time }</dd>

    <!-- Add: Time, author, lines edited -->
</dl>

<!-- <div class="slider-container">
    <div class="slider">
        <label for="slider">Show commits until:</label>
        <input type="range" id="slider" name="slider" min=0 max=100 bind:value={commitProgress}/>
    </div>
    <time class="time-label">{commitMaxTime.toLocaleString()}</time>
</div> -->

<Scrolly bind:progress={ commitProgress }>
	{#each commits as commit, index }
        <p>
            On {commit.datetime.toLocaleString("en", {dateStyle: "full", timeStyle: "short"})},
            {index === 0 
                ? "I set forth on my very first commit, beginning a magical journey of code. You can view it "
                : "I added another commit. See it "}
            <a href="{commit.url}" target="_blank">
                {index === 0 ? "here" : "here"}
            </a>.
            This update transformed {commit.totalLines} lines across { d3.rollups(commit.lines, D => D.length, d => d.file).length } files.
            With every commit, our project grows.
        </p>
    {/each}
	<svelte:fragment slot="viz">
		<svg viewBox="0 0 {width} {height}">
            <g class="dots">
                {#each filteredCommits as commit, index }
                    <circle
                        class:selected={ clickedCommits.includes(commit) }
                        on:click={ evt => dotInteraction(index, evt) }
                        on:mouseenter={evt => dotInteraction(index, evt)}
                        on:mouseleave={evt => dotInteraction(index, evt)}
                        cx={ xScale(commit.datetime) }
                        cy={ yScale(commit.hourFrac) }
                        r={ rScale(commit.totalLines) }
                        fill="steelblue"
                        fill-opacity="0.5"
                    />
                {/each}
            </g>
            <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
            <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />
            <g class="gridlines" transform="translate({usableArea.left}, 0)" bind:this={yAxisGridlines} />
        </svg>
	</svelte:fragment>
</Scrolly>

<Scrolly bind:progress={ commitProgress }>
	{#each commits as commit, index }
        <p>
            On {commit.datetime.toLocaleString("en", {dateStyle: "full", timeStyle: "short"})},
            {index === 0 
                ? "I set forth on my very first commit, beginning a magical journey of code. You can view it "
                : "I added another commit. See it "}
            <a href="{commit.url}" target="_blank">
                {index === 0 ? "here" : "here"}
            </a>.
            This update transformed {commit.totalLines} lines across { d3.rollups(commit.lines, D => D.length, d => d.file).length } files.
            With every commit, our project grows.
        </p>
    {/each}
	<svelte:fragment slot="viz">
		<FileLines lines={filteredData} width={width}/>
	</svelte:fragment>
</Scrolly>

<Bar data={languageBreakdown} width={width} />

<p>Total lines of code: {data.length}</p>

<section>
    <h2>Summary</h2>
    <dl class="stats">
        <dt>Total <abbr title="Lines of code">LOC</abbr></dt>
        <dd>{data.length}</dd>
        <dt>Files</dt>
        <dd>{d3.groups(filteredData, d => d.file).length}</dd>
        <dt>Commits</dt>
        <dd>{d3.groups(filteredData, d => d.commit).length}</dd>
    </dl>
</section>

<style>
    dl{
        display: grid;
        grid-template-columns: auto;
    }
    dt{
        grid-row: 1;
        font-family: inherit;
        font-weight: bold;
        color: var(--border-gray);
        text-transform: uppercase;
    }
    dd{
        font-family: inherit;
        font-weight: bold;
    }
    section{
        border-width:0.15em;
        border-style:solid;
        border-color:var(--border-gray);
        padding-left: 1em;
        padding-right: 1em;
    }
    svg {
        overflow: visible;
    }
    .gridlines {
        stroke-opacity: .2;
    }
    .info{
        display: grid;
        margin:0;
        grid-template-columns: 2;
    }

    .info dt{
        grid-column:1;
        grid-row:auto;
    }

    .info dd{
        grid-column:2;
        grid-row:auto;
        font-weight: 400;
    }

    .tooltip{
        position: fixed;
        top: 1em;
        left: 1em;
    }
    circle {
        transition: 200ms;

        transform-origin: center;
        
        transform-box: fill-box;

        &:hover {
            transform: scale(1.5);
        }

        @starting-style {
            r: 0;
        }
    }
    .info{
        display: grid;
        margin:0;
        grid-template-columns: 2;
        background-color: oklch(100% 0% 0 / 80%);
        box-shadow: 1px 1px 3px 3px gray;
        border-radius: 5px;
        backdrop-filter: blur(10px);
        padding:10px;
        transition-duration: 500ms;
        transition-property: opacity, visibility;

        &[hidden]:not(:hover, :focus-within) {
            opacity: 0;
            visibility: hidden;
        }
    }

    .selected {
        fill: var(--color-accent);
    }
    .slider-container{
        display:grid;
    }
    .slider{
        display: flex;
    }
    #slider{
        flex:1;
    }
    .time-label{
        font-size: 0.75em;
        text-align: right;
    }
</style>