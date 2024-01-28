<script>
	import { onMount } from 'svelte';
	
	onMount(async () => {
		miro.board.ui.on('icon:click', async () => {
			await miro.board.ui.openPanel({url: 'app'});
		});
	});
</script>

<svelte:head>
	<title>Home</title>
	<meta name="description" content="Svelte demo app" />
	<script src="https://miro.com/app/static/sdk/v2/miro.js"></script>
</svelte:head>

<section>
	<div class="header">
		<p class="title">Miro CPD Svelte</p>
		<p class="subtitle">Interface for Miro CPD</p>
	</div>
	<div class="form">
	</div>
</section>

<style>
	.title {
		font-size: 1.2rem;
		font-weight: 600;
	}
	.subtitle {
		font-size: .9rem;
		font-weight: 300;
		padding-top: 10px
	}
	.heading {
		font-weight: 400;
		font-size: .9rem;
		padding-bottom: 5px;
	}
	.header {
		background-color: #2B3F88;
		width: 100%;
		padding: 1.5rem 0;
		text-align: center;
		color: white
	}
	.heading_unit {
		font-weight: 300;
		font-size: 90%;
		float: right;
		padding-top: 3px;
	}
	.form {
		padding: 2rem;
		margin-bottom: 1rem;
	}
	.unit-main {
		margin-bottom: 2rem;
	}

	h1 {
		width: 100%;
	}
</style>
