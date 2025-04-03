<script>
    import projects from "$lib/projects.json";
    import Project from "$lib/Project.svelte";
    import Pie from '$lib/Pie.svelte';
    import * as d3 from 'd3';

    let query = "";
    
    let selectedYearIndex = -1;
    let selectedYear;
    $: selectedYear = selectedYearIndex > -1 ? pieData[selectedYearIndex].label : null;
    $: filteredProjects = projects.filter(project => {
    let values = Object.values(project).join("\n").toLowerCase();
    return values.includes(query.toLowerCase());
    });

    $: filteredByYear = selectedYear ? filteredProjects.filter(project => project.year === selectedYear) : filteredProjects;

    /*$: rolledData = d3.rollups(filteredProjects, v => v.length, d => d.year);

    // let pieData = [
    // { value: 1, label: "apples" },
    // { value: 2, label: "oranges" },
    // { value: 3, label: "mangos" },
    // { value: 4, label: "pears" },
    // { value: 5, label: "limes" },
    // { value: 5, label: "cherries" }
    // ];
    $: pieData = rolledData.map(([year, count]) => {
      return { value: count, label: year };
    });*/
    let pieData;

    $: {
        // Initialize to an empty object every time this runs
        pieData = {};
        
        // Calculate rolledData and pieData based on filteredProjects here
        let rolledData = d3.rollups(filteredProjects, v => v.length, d => d.year);

        // We don't need 'let' anymore since we already defined pieData
        pieData = rolledData.map(([year, count]) => {
            return { value: count, label: year };
        });
    }

</script>

<svelte:head>
  <title>Projects</title>
</svelte:head>

<Pie data={pieData}  bind:selectedIndex={selectedYearIndex} />
<input type="search" bind:value={query} aria-label="Search projects" placeholder="🔍 Search projects..." />

<h1>{ projects.length } Projects</h1>
<div class="projects">
    {#each filteredByYear as fp}
        <Project data={fp}/>
      <!-- <article>
        <h2>{p.title}</h2>
        <img src={p.image} alt="" />
        <p>
            {p.description}
        </p>
      </article> -->
    {/each}
</div>