<!--
	@component
	This is the main editor component that combines the input and renderer
	components. It also handles the scroll synchronization between the input and renderer
	components (if set to sync), and the window mode management (tabs or split).
-->

<script lang="ts">
	import type { Carta } from './internal/carta';
	import type { TextAreaProps } from './internal/textarea-props';
	import { onMount } from 'svelte';
	import { defaultLabels, type Labels } from './internal/labels';
	import Renderer from './internal/components/Renderer.svelte';
	import Input from './internal/components/Input.svelte';
	import Toolbar from './internal/components/Toolbar.svelte';

	interface Props {
		/**
		 * The Carta instance to use.
		 */
		carta: Carta;
		/**
		 * The theme to use, which translates to the CSS class `carta-theme__{theme}`.
		 */
		theme?: string;
		/**
		 * The editor content.
		 */
		value?: string;
		/**
		 * The mode to use. Can be 'tabs', 'split', or 'auto'
		 * - 'tabs': The input and renderer are displayed in tabs.
		 * - 'split': The input and renderer are displayed side by side.
		 * - 'auto': The mode is automatically selected based on the window width.
		 */
		mode?: 'tabs' | 'split' | 'auto';
		/**
		 * The scroll synchronization mode. Can be 'sync' or 'async'.
		 * - 'sync': The scroll is synchronized between the input and renderer.
		 * - 'async': The scroll is not synchronized between the input and renderer.
		 */
		scroll?: 'sync' | 'async' | 'cursor';
		/**
		 * Whether to disable the toolbar.
		 */
		disableToolbar?: boolean;
		/**
		 * The placeholder text for the textarea.
		 */
		placeholder?: string;
		/**
		 * Additional textarea properties.
		 */
		textarea?: TextAreaProps;
		/**
		 * The selected tab. Can be 'write' or 'preview'.
		 */
		selectedTab?: 'write' | 'preview';
		/**
		 * The labels to use for the editor.
		 */
		userLabels?: Partial<Labels>;
		/**
		 * Highlight delay in milliseconds.
		 * @default 250
		 */
		highlightDelay?: number;
	}

	let {
		carta,
		theme = 'default',
		value = $bindable(''),
		mode = 'auto',
		scroll = 'sync',
		disableToolbar = false,
		placeholder = '',
		textarea = {},
		selectedTab = 'write',
		userLabels = {},
		highlightDelay = 250
	}: Props = $props();

	const labels = $derived({
		...defaultLabels,
		...userLabels
	});

	let width = $state(0);
	let mounted = $state(false);

	let editorElem: HTMLDivElement;
	let inputElem: HTMLDivElement | undefined = $state();
	let rendererElem: HTMLDivElement | undefined = $state();
	let input: Input | undefined = $state();

	// Change the window mode based on the width
	const windowMode: 'tabs' | 'split' = $derived(
		mode === 'auto' ? (width && width > 768 ? 'split' : 'tabs') : mode
	);

	$effect(() => {
		windowMode; // Resize when changing from tabs to split
		input && input.resize();
	});

	$effect(() => {
		inputElem;
		rendererElem;
		if (windowMode === 'tabs') setScrollPosition();
	});

	let currentScrollLine = 1;
	let isSyncingMarkdown = false;
	let isSyncingHtml = false;

	const SMOOTH_SCROLL_DURATION = 500;
	const SYNC_DELAY = 150;

	// Store ongoing animation frame IDs to make smooth scroll interruptible
	const animationFrameIds = new Map();

	/**
	 * Custom smooth scroll function.
	 * @param {HTMLDivElement} element The scrollable element.
	 * @param {number} to The target scrollTop value.
	 * @param {number} duration The duration of the scroll in milliseconds.
	 * @param {function} onComplete Callback when scroll is complete.
	 */
	function customSmoothScroll(element:HTMLDivElement, to:number, duration:number, onComplete:() => void) {

		// If there's an existing animation for this element, cancel it
		if (animationFrameIds.has(element)) {
			cancelAnimationFrame(animationFrameIds.get(element));
			animationFrameIds.delete(element);
		}

		const start = element.scrollTop;
		const change = to - start;
		const startTime = performance.now();

		// Easing function: easeInOutQuad
		// t: current time, b: beginning value, c: change in value, d: duration
		const easeInOutQuad = (t:number, b:number, c:number, d:number) => {
			t /= d / 2;
			if (t < 1) return c / 2 * t * t + b;
			t--;
			return -c / 2 * (t * (t - 2) - 1) + b;
		};

		function animateScroll() {
			const currentTime = performance.now();
			const elapsed = currentTime - startTime;

			element.scrollTop = easeInOutQuad(elapsed, start, change, duration);

			if (elapsed < duration) {
				const frameId = requestAnimationFrame(animateScroll);
				animationFrameIds.set(element, frameId);
			} else {
				// Ensure final position is exact
				element.scrollTop = to;
				animationFrameIds.delete(element);
				if (onComplete) {
					onComplete();
				}
			}
		}

		animateScroll();
	}

	/**
	 * Finds the HTML element with a matching data-line attribute, or the first element before it.
	 * @param targetLineNumber
	 */
	function findElementByLineOrBefore(targetLineNumber: number): HTMLElement | null {
		if(!rendererElem) return null;

		const exactMatch = rendererElem.querySelector(`[data-line="${currentScrollLine}"]`);
		if (exactMatch instanceof HTMLElement) {
			return exactMatch;
		}

		const elements = rendererElem.querySelectorAll('[data-line]');
		let bestMatch: HTMLElement | null = null;
		let bestMatchLineValue = -Infinity;

		// iterate to find the closest match before
		elements.forEach(element => {
			if (element instanceof HTMLElement) {
				const lineValue = parseInt(element.dataset.line!, 10);
				if (!isNaN(lineValue)) {
					if (lineValue < targetLineNumber && lineValue > bestMatchLineValue) {
						bestMatch = element;
						bestMatchLineValue = lineValue;
					}
				}
			}
		});

		return bestMatch;
	}

	/**
	 * Set the scroll position of the active tab based on the current line.
	 */
	function setScrollPosition() {
		if(selectedTab === 'write') {
			if (!inputElem) return;

			const markdownLineElements = inputElem
					.querySelector('.carta-highlight')
					?.querySelector('pre > code')
					?.querySelectorAll<HTMLElement>('span.line');

			if (!markdownLineElements || currentScrollLine > markdownLineElements.length) {
				return;
			}

			const mdElem = markdownLineElements[currentScrollLine - 1];
			if(mdElem) {
				isSyncingMarkdown = true;
				mdElem.scrollIntoView({ block: 'start', behavior: 'instant' });
				setTimeout(() => { isSyncingMarkdown = false; }, SYNC_DELAY);
			}
		} else {
			if(!rendererElem) return;

			const htmlElem = findElementByLineOrBefore(currentScrollLine);
			if (htmlElem) {
				isSyncingHtml = true;
				htmlElem.scrollIntoView({ block: 'start', behavior: 'instant' });
				setTimeout(() => { isSyncingHtml = false; }, SYNC_DELAY);
			}
		}
	}

	/**
	 * Synchronizes scrolling between Input and Renderer.
	 *
	 * Uses IntersectionObserver to identify markdown lines (`span.line` by index)
	 * and HTML lines (`[data-line]` attribute) and keep their scroll positions aligned.
	 * Uses MutationObserver to handle dynamic content.
	 *
	 * @param {HTMLDivElement} markdownPane The scrollable markdown editor container.
	 * @param {HTMLDivElement} htmlPane The scrollable HTML preview container.
	 * @returns {function} A cleanup function to disconnect observers.
	 */
	function setupScrollSync(markdownPane:HTMLDivElement, htmlPane:HTMLDivElement): () => void {

		const highlightContainer = markdownPane.querySelector('.carta-highlight');
		if (!highlightContainer) { console.error("no highlightContainer"); return () => {}; }

		const getMarkdownCodeElement = () => highlightContainer.querySelector('pre > code');

		const observerOptions = (rootElement: HTMLElement) => ({
			root: rootElement,
			rootMargin: '0px 0px -95% 0px',
			threshold: 0
		});

		const getTopmostVisibleEntry = (entries: IntersectionObserverEntry[]) => {
			const visibleEntries = entries.filter(e => e.isIntersecting && e.target.isConnected);
			if (!visibleEntries.length) return null;
			return visibleEntries.reduce((prev, curr) =>
					prev.boundingClientRect.top < curr.boundingClientRect.top ? prev : curr
			);
		};

		let firstHTMLIntersection = true;

		// --- HTML (Rendered View) to Markdown (Editor) Sync ---
		const htmlObserver = new IntersectionObserver((entries) => {
			// When we initialize, some elements will already be intersecting. We don't want to trigger the scroll
			// for those, so we skip the first intersection.
			if(firstHTMLIntersection) {
				firstHTMLIntersection = false;
				return;
			}

			if (isSyncingHtml) return;

			const topEntry = getTopmostVisibleEntry(entries);
			if (!topEntry) return;

			const lineNumStr = topEntry.target.getAttribute('data-line');
			if (!lineNumStr) return;

			currentScrollLine = parseInt(lineNumStr, 10);

			const codeElement = getMarkdownCodeElement();
			if (!codeElement) return;

			const markdownLines = codeElement.querySelectorAll('span.line');
			const mdElem = markdownLines[currentScrollLine - 1];

			if (mdElem) {
				isSyncingMarkdown = true;

				const targetScrollTop = mdElem.getBoundingClientRect().top -
						markdownPane.getBoundingClientRect().top +
						markdownPane.scrollTop;

				customSmoothScroll(markdownPane, targetScrollTop, SMOOTH_SCROLL_DURATION, () => {
					isSyncingMarkdown = false;
				});
			}

		}, observerOptions(htmlPane));

		// --- Markdown (Editor) to HTML (Rendered View) Sync ---
		const markdownObserver = new IntersectionObserver((entries) => {
			if (isSyncingMarkdown) return;

			const topEntry = getTopmostVisibleEntry(entries);
			if (!topEntry) return;

			const codeElement = getMarkdownCodeElement();
			if (!codeElement || !codeElement.contains(topEntry.target)) return;

			const allMarkdownLines = Array.from(codeElement.children);
			const lineIndex = allMarkdownLines.indexOf(topEntry.target);
			if (lineIndex === -1) return;
			currentScrollLine = lineIndex + 1;

			const htmlElem = htmlPane.querySelector(`[data-line="${currentScrollLine}"]`);

			if (htmlElem) {
				isSyncingHtml = true;
				const targetScrollTop = htmlElem.getBoundingClientRect().top -
						htmlPane.getBoundingClientRect().top +
						htmlPane.scrollTop;
				customSmoothScroll(htmlPane, targetScrollTop, SMOOTH_SCROLL_DURATION, () => {
					isSyncingHtml = false;
				});
			}

		}, observerOptions(markdownPane));


		function observeAll() {
			markdownObserver.disconnect();
			htmlObserver.disconnect();

			// Observe all markdown lines
			const codeElement = getMarkdownCodeElement();
			if (codeElement) {
				codeElement.querySelectorAll('span.line').forEach(el => markdownObserver.observe(el));
			}

			// Observe all HTML elements with data-line attribute
			htmlPane.querySelectorAll('[data-line]').forEach(el => htmlObserver.observe(el));
			firstHTMLIntersection = true;
		}

		let observeAllTimeoutId: number | null = null;
		const DEBOUNCE_OBSERVE_ALL_DELAY = 150;
		function debouncedObserveAll(): void {
			if (observeAllTimeoutId !== null) {
				clearTimeout(observeAllTimeoutId);
			}
			observeAllTimeoutId = window.setTimeout(() => {
				observeAll();
				observeAllTimeoutId = null;
			}, DEBOUNCE_OBSERVE_ALL_DELAY);
		}

		requestAnimationFrame(observeAll);

		const mutationObserverConfig = { childList: true, subtree: true };
		const mdMutationObserver = new MutationObserver(debouncedObserveAll);
		mdMutationObserver.observe(highlightContainer, mutationObserverConfig);

		const htmlMutationObserver = new MutationObserver(debouncedObserveAll);
		htmlMutationObserver.observe(htmlPane, mutationObserverConfig);

		return () => {
			// Clean up observers
			markdownObserver.disconnect();
			htmlObserver.disconnect();
			mdMutationObserver.disconnect();
			htmlMutationObserver.disconnect();
			// Clear any pending animations
			if (animationFrameIds.has(markdownPane)) {
				cancelAnimationFrame(animationFrameIds.get(markdownPane));
				animationFrameIds.delete(markdownPane);
			}
			if (animationFrameIds.has(htmlPane)) {
				cancelAnimationFrame(animationFrameIds.get(htmlPane));
				animationFrameIds.delete(htmlPane);
			}
		};
	}

	onMount(() => carta.$setElement(editorElem));
	onMount(() => (mounted = true));
	onMount(() => {
		if(scroll === 'sync') {
			if(inputElem && rendererElem) {
				setupScrollSync(inputElem, rendererElem)
			} else {
				console.error("Input or Renderer element is not defined");
			}
		}
	});

</script>

<div bind:this={editorElem} bind:clientWidth={width} class="carta-editor carta-theme__{theme}">
	{#if !disableToolbar}
		<Toolbar {carta} {labels} mode={windowMode} bind:tab={selectedTab} />
	{/if}

	<div class="carta-wrapper">
		<div class="carta-container mode-{windowMode}">
			<Input
				{carta}
				{placeholder}
				{highlightDelay}
				props={textarea}
				hidden={!(windowMode === 'split' || selectedTab === 'write')}
				bind:value
				bind:this={input}
				bind:elem={inputElem}
			>
				<!-- Input extensions components -->
				{#if mounted}
					{#each carta.components.filter(({ parent }) => [parent]
							.flat()
							.includes('input')) as { component: DynamicComponent, props }}
						<DynamicComponent {carta} {...props}></DynamicComponent>
					{/each}
				{/if}
			</Input>
			<Renderer
				{carta}
				{value}
				hidden={!(windowMode === 'split' || selectedTab === 'preview')}
				bind:elem={rendererElem}
			>
				<!-- Renderer extensions components -->
				{#if mounted}
					{#each carta.components.filter(({ parent }) => [parent]
							.flat()
							.includes('renderer')) as { component: DynamicComponent, props }}
						<DynamicComponent {carta} {...props}></DynamicComponent>
					{/each}
				{/if}
			</Renderer>
		</div>
	</div>

	<!-- Editor extensions components -->
	<!-- prevent loading components on ssr renderings -->
	{#if mounted}
		{#each carta.components.filter(({ parent }) => [parent]
				.flat()
				.includes('editor')) as { component: DynamicComponent, props }}
			<DynamicComponent {carta} {...props}></DynamicComponent>
		{/each}
	{/if}
</div>

<style>
	.carta-editor {
		position: relative;
		display: flex;
		flex-direction: column;
	}

	:global(.carta-container.mode-split > *) {
		width: 50%;
	}

	:global(.carta-container.mode-tabs > *) {
		width: 100%;
	}

	.carta-container {
		display: flex;
		position: relative;
	}
</style>
