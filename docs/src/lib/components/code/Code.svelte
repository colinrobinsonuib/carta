<script lang="ts">
	import type { HighlightedCodeBlock } from '.';

	interface Props {
		code: HighlightedCodeBlock;
	}

	let { code }: Props = $props();
	let elem: HTMLElement | undefined = $state();
</script>

<div bind:this={elem} class="code-block relative">
	<!-- eslint-disable-line svelte/no-at-html-tags -->{@html code.html}

	<!-- Copy button -->
	<button
		aria-label="Copy code"
		title="Copy"
		onclick={() => {
			navigator.clipboard.writeText(elem!.innerText);
		}}
		class="
      absolute right-4 top-[min(50%_,_32px)] aspect-square -translate-y-1/2 transform
      rounded hover:bg-neutral-800 hover:text-neutral-300 active:text-sky-300
    "
	>
		<iconify-icon icon="octicon:copy-16" class="p-2 text-lg"></iconify-icon>
	</button>
</div>
