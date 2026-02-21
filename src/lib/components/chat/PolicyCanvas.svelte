<script lang="ts">
	import { Pane, PaneResizer } from 'paneforge';
	import { onDestroy, onMount } from 'svelte';

	import { showPolicyCanvas, policyCanvasSessionId } from '$lib/stores';
	import Drawer from '../common/Drawer.svelte';
	import PolicyCanvasContent from './PolicyCanvasContent.svelte';

	// ── Props ──────────────────────────────────────────────────────────────
	export let pane;
	export let sessionId = null;
	export let onProceed = null;
	export let apiBase = 'http://localhost:8000';

	// ── Pane layout (mirrors ChatControls) ────────────────────────────────
	let mediaQuery;
	let largeScreen = false;
	let minSize = 0;

	// ── Pane helpers ──────────────────────────────────────────────────────
	export const openPane = () => {
		if (!pane) return;
		if (parseInt(localStorage?.policyCanvasSize)) {
			const container = document.getElementById('chat-container');
			if (container) {
				let size = Math.floor(
					(parseInt(localStorage?.policyCanvasSize) / container.clientWidth) * 100
				);
				pane.resize(size);
			}
		} else {
			pane.resize(minSize);
		}
	};

	const handleMediaQuery = async (e) => {
		if (e.matches) {
			largeScreen = true;
		} else {
			largeScreen = false;
			pane = null;
		}
	};

	let resizeObserver;

	// ── Lifecycle ─────────────────────────────────────────────────────────
	onMount(() => {
		mediaQuery = window.matchMedia('(min-width: 1024px)');
		mediaQuery.addEventListener('change', handleMediaQuery);
		handleMediaQuery(mediaQuery);

		const container = document.getElementById('chat-container');
		if (container) {
			// initialize the minSize based on the container width
			minSize = Math.floor((350 / container.clientWidth) * 100);

			// Create a ResizeObserver to dynamically recalculate minSize
			resizeObserver = new ResizeObserver((entries) => {
				for (let entry of entries) {
					const width = entry.contentRect.width;
					// calculate the percentage of 350px
					const percentage = (350 / width) * 100;
					// set the minSize to the percentage, must be an integer
					minSize = Math.floor(percentage);

					if ($showPolicyCanvas) {
						if (pane && pane.isExpanded() && pane.getSize() < minSize) {
							pane.resize(minSize);
						} else {
							let size = Math.floor(
								(parseInt(localStorage?.policyCanvasSize) / container.clientWidth) * 100
							);
							if (pane && size < minSize) {
								pane.resize(minSize);
							}
						}
					}
				}
			});

			// Start observing the container's size changes
			resizeObserver.observe(container);
		}

	});

	onDestroy(() => {
		showPolicyCanvas.set(false);

		if (resizeObserver) {
			resizeObserver.disconnect();
		}
		if (mediaQuery) {
			mediaQuery.removeEventListener('change', handleMediaQuery);
		}
	});

	$: if (sessionId) {
		policyCanvasSessionId.set(sessionId);
	}
</script>

{#if !largeScreen}
	{#if $showPolicyCanvas}
		<Drawer
			show={$showPolicyCanvas}
			onClose={() => {
				showPolicyCanvas.set(false);
			}}
		>
			<div class="h-full bg-white dark:bg-gray-900 dark:text-gray-100">
				<PolicyCanvasContent {sessionId} {onProceed} {apiBase} />
			</div>
		</Drawer>
	{/if}
{:else}
	{#if $showPolicyCanvas}
		<PaneResizer
			class="relative flex items-center justify-center group border-l border-gray-50 dark:border-gray-850/30 hover:border-gray-200 dark:hover:border-gray-800 transition z-20"
			id="policy-canvas-resizer"
		>
			<div
				class="absolute -left-1.5 -right-1.5 -top-0 -bottom-0 z-20 cursor-col-resize bg-transparent"
			/>
		</PaneResizer>
	{/if}

	<Pane
		bind:pane
		defaultSize={0}
		onResize={(size) => {
			if ($showPolicyCanvas && pane && pane.isExpanded()) {
				if (size < minSize) {
					pane.resize(minSize);
				}

				if (size < minSize) {
					localStorage.policyCanvasSize = 0;
				} else {
					const container = document.getElementById('chat-container');
					if (container) {
						localStorage.policyCanvasSize = Math.floor(
							(size / 100) * container.clientWidth
						);
					}
				}
			}
		}}
		onCollapse={() => {
			showPolicyCanvas.set(false);
		}}
		collapsible={true}
		class="z-10 bg-white dark:bg-gray-850"
	>
		{#if $showPolicyCanvas}
			<div class="flex max-h-full min-h-full">
				<div
					class="w-full bg-white dark:shadow-lg dark:bg-gray-850 z-40 pointer-events-auto overflow-y-auto scrollbar-hidden"
					id="policy-canvas-container"
				>
					<PolicyCanvasContent {sessionId} {onProceed} {apiBase} />
				</div>
			</div>
		{/if}
	</Pane>
{/if}
