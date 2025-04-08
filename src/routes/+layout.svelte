<style>
nav ul,
nav il {
	display: contents; /* step 2.1 */
}

nav {
	display: flex;
	margin-bottom: 1em;
	border-bottom-width: 1px;
	border-bottom-style: solid;
	border-bottom-color: black;
}

nav a {
	flex: 1;
	text-decoration: none;
	color: inherit;
	text-align: center;
	padding: 0.1em;
}

nav a.current {
	border-bottom-width: 0.4em; 
	border-bottom-style: solid;
	border-bottom-color: var(--gray);
	padding-bottom:0.1em;
	font-weight: bold;
}

nav a:hover {
	background-color: var(--gray);
	font-weight: bold;
	margin: 0 auto;
	
	border-bottom-width: 0.4em; 
	border-bottom-style: solid;
	border-bottom-color: var(--color-acent);
	padding: -0.1em;
}
</style>

<script>
import { base } from "$app/paths";
import { page } from "$app/stores";

let pages = [
    { url: "/", title: "Home" },
    { url: "/projects", title: "Projects" },
    { url: "/contact", title: "Contact" },
    { url: "/cv", title: "Resume" },
    { url: "/meta", title: "Meta" },
    {url: "https://github.com/GuilhermeCastilho02", title:"Github"}
];

let localStorage = globalThis.localStorage ?? {};

let colorScheme = localStorage.colorScheme ?? "light dark";
let root = globalThis?.document?.documentElement;
$: root?.style.setProperty("color-scheme", colorScheme);
$: localStorage.colorScheme = colorScheme;

</script>

<nav>
  {#each pages as p}
    <a
      href={p.url.startsWith("http") ? p.url: `${base}${p.url}`}
      class:current={$page.route.id === p.url}
    >
      {p.title}
    </a>
  {/each}
</nav>

<label class="color-scheme">
    Theme:
    <select bind:value={ colorScheme }>
        <option value="light dark"> Automatic </option>
        <option value="light"> Light </option>
        <option value="dark"> Dark </option>
    </select>
</label>

<slot />
