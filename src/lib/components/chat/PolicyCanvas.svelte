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

	// ── Pane layout (mirrors ChatControls) ────────────────────────────────
	let mediaQuery;
	let largeScreen = false;
	let dragged = false;
	let minSize = 0;

	// ── Pane helpers ──────────────────────────────────────────────────────
	export const openPane = () => {
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

	const onMouseDown = () => {
		dragged = true;
	};

	const onMouseUp = () => {
		dragged = false;
	};

	// ── Lifecycle ─────────────────────────────────────────────────────────
	onMount(() => {
		mediaQuery = window.matchMedia('(min-width: 1024px)');
		mediaQuery.addEventListener('change', handleMediaQuery);
		handleMediaQuery(mediaQuery);

		const container = document.getElementById('chat-container');
		if (container) {
			minSize = Math.floor((350 / container.clientWidth) * 100);
		}

		document.addEventListener('mousedown', onMouseDown);
		document.addEventListener('mouseup', onMouseUp);
	});

	onDestroy(() => {
		showPolicyCanvas.set(false);

		if (mediaQuery) {
			mediaQuery.removeEventListener('change', handleMediaQuery);
		}
		document.removeEventListener('mousedown', onMouseDown);
		document.removeEventListener('mouseup', onMouseUp);
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
				<PolicyCanvasContent {sessionId} {onProceed} />
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
					<PolicyCanvasContent {sessionId} {onProceed} />
				</div>
			</div>
		{/if}
	</Pane>
{/if}
