<script lang="ts">
	import { Spinner } from 'flowbite-svelte';
	import { fade, fly } from 'svelte/transition';

	interface Props {
		show?: boolean;
		loadingMsg?: string;
	}

	let { show = true, loadingMsg = 'Loading...' }: Props = $props();
</script>

{#if show}
	<div class="flex justify-center">
		<div class="fixed top-6 bg-gray-900 rounded-lg opacity-90 z-10" in:fade out:fly={{ y: '-50%' }}>
			<Spinner class="m-4" />
			<span class="dark:text-white mr-4">{loadingMsg}</span>
		</div>
	</div>
{/if}
