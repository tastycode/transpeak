<script>
	import Nav from '../components/Nav.svelte';
</script>

<style>
    @media only screen and (min-width: 460px) {
        main {
            max-width: 55em;
            margin: 0 auto;
        }
    }
	main {
		position: relative;
        margin: 1.5em;
        overflow-x: hidden;
        padding-top: 1em;
        padding-bottom:1em;
		box-sizing: border-box;
	}
</style>


<main>
	<slot></slot>
</main>
