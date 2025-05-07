<script lang="ts">
	import { MarkdownEditor } from '$lib';
	import { Carta, type Plugin } from '$lib/internal/carta';
	import type { Root, Element } from 'hast';
	import { visit } from 'unist-util-visit';
	import ToggleTheme from './ToggleTheme.svelte';
	import sampleText from './sample.md?raw';
	import '$lib/default.css';

	export const betterscroll = (): Plugin => {
		return {
			transformers: [
				{
					execution: 'sync',
					type: 'rehype',
					transform: ({ processor }) => {
						processor.use(rehypeLineNumbers);
					},
				},
			],
		};
	};

	function rehypeLineNumbers() {
		return (tree: Root) => {
			visit(tree, 'element', (node: Element, index, parent) => {
				// Only apply to elements that are direct children of the root
				if (parent === tree && node.position?.start?.line) {
					if (!node.properties) {
						node.properties = {};
					}
					node.properties['data-line'] = node.position.start.line;
				}
			});
		};
	}

	const carta = new Carta({
		extensions: [betterscroll()],
	});
</script>

<svelte:head>
	<!-- Custom fonts -->
	<!-- Fira font -->
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Fira+Code:wght@300;400;500;600;700&display=swap"
		rel="stylesheet"
	/>
	<!-- Inter -->
	<link rel="preconnect" href="https://rsms.me/" />
	<link rel="stylesheet" href="https://rsms.me/inter/inter.css" />
</svelte:head>

<main>
	<ToggleTheme class="toggle-theme" />
	<MarkdownEditor scroll="click" value={sampleText} placeholder="Some text..." mode="split" {carta} />
</main>

<style>
	:global(body) {
		margin: 0;
		font-family: 'Inter var', sans-serif;
		min-height: 100vh;
	}

	:global(.carta-font-code) {
		font-family: 'Fira Code', monospace;
		font-variant-ligatures: normal;
		font-size: 1.1rem;
	}

	:global(input, textarea, button) {
		font-family: inherit;
	}

	main {
		position: relative;

		max-width: 1536px;
		margin: 2rem auto 2rem auto;
	}

	:global(img) {
		max-width: 50%;
	}

	:global(.toggle-theme) {
		position: absolute;
		top: 0;
		left: -6px;
		transform: translateX(-100%);
	}

	/* Responsive main */

	@media screen and (max-width: 640px) {
		main {
			width: 95%;
		}
	}

	@media screen and (min-width: 640px) and (max-width: 767px) {
		main {
			width: 640px;
		}
	}

	@media screen and (min-width: 767px) and (max-width: 1023px) {
		main {
			width: 768px;
		}
	}

	@media screen and (min-width: 1023px) and (max-width: 1279px) {
		main {
			width: 1024px;
		}
	}

	@media screen and (min-width: 1279px) and (max-width: 1535px) {
		main {
			width: 1280px;
		}
	}
</style>
