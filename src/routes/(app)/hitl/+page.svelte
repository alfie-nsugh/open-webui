<script lang="ts">
	import { onMount } from 'svelte';
	import { toast } from 'svelte-sonner';

	const API_BASE = `http://${window.location.hostname}:8000`;

	type AxiomItem = {
		symbol: string;
		plain_comment: string;
		lean_decl: string;
	};

	type ProofDetail = {
		stage_id: string;
		item_type: 'new_proof' | 'reuse_check' | 'unprovable';
		group_id: string;
		original_question: string;
		agent_reasoning: string;
		submitted_at: string;
		status: string;
		lean_body: string | null;
		provenance_text: string;
		new_axioms: AxiomItem[];
		new_defs: AxiomItem[];
		theorem: AxiomItem | null;
		reused_imports: string[];
		unclear_aspect: string;
		decision_notes: string;
		clarification: string | null;
	};

	let items: ProofDetail[] = [];
	let loading = true;
	let refreshing = false;

	async function loadItems() {
		refreshing = true;
		try {
			const res = await fetch(`${API_BASE}/hitl/pending`);
			if (!res.ok) throw new Error(await res.text());
			const summaries: { stage_id: string }[] = await res.json();

			const details = await Promise.all(
				summaries.map(async (s) => {
					const r = await fetch(`${API_BASE}/hitl/${s.stage_id}`);
					return r.ok ? (r.json() as Promise<ProofDetail>) : null;
				})
			);
			items = details.filter(Boolean) as ProofDetail[];
		} catch (e: any) {
			toast.error(`Failed to load proof history: ${e.message}`);
		} finally {
			loading = false;
			refreshing = false;
		}
	}

	onMount(() => {
		loadItems();
	});

	function formatDate(iso: string): string {
		return new Date(iso).toLocaleString();
	}

	function statusBadgeClass(status: string): string {
		switch (status) {
			case 'approved':
			case 'approved_with_edit':
				return 'bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-200';
			case 'rejected':
				return 'bg-red-100 text-red-800 dark:bg-red-900 dark:text-red-200';
			default:
				return 'bg-gray-100 text-gray-800 dark:bg-gray-700 dark:text-gray-300';
		}
	}
</script>

<svelte:head>
	<title>Proof History — BasedQED</title>
</svelte:head>

<div class="w-full max-w-4xl mx-auto p-6 pb-20 flex-1 overflow-y-auto">
	<div class="flex items-center justify-between">
		<h1 class="text-2xl font-bold dark:text-white">Proof History</h1>
		<button
			class="flex items-center gap-2 px-3 py-1.5 text-sm rounded-lg border dark:border-gray-700 hover:bg-gray-100 dark:hover:bg-gray-800 transition"
			on:click={loadItems}
			disabled={refreshing}
		>
			<svg xmlns="http://www.w3.org/2000/svg" class="size-4 {refreshing ? 'animate-spin' : ''}" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor">
				<path stroke-linecap="round" stroke-linejoin="round" d="M16.023 9.348h4.992v-.001M2.985 19.644v-4.992m0 0h4.992m-4.993 0 3.181 3.183a8.25 8.25 0 0 0 13.803-3.7M4.031 9.865a8.25 8.25 0 0 1 13.803-3.7l3.181 3.182m0-4.991v4.99" />
			</svg>
			Refresh
		</button>
	</div>

	{#if loading}
		<div class="flex items-center justify-center py-20 text-gray-500 dark:text-gray-400">
			Loading proof history...
		</div>
	{:else if items.length === 0}
		<div class="flex flex-col items-center justify-center py-20 text-gray-500 dark:text-gray-400 gap-2">
			<svg xmlns="http://www.w3.org/2000/svg" class="size-12 opacity-40" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor">
				<path stroke-linecap="round" stroke-linejoin="round" d="M9 12.75 11.25 15 15 9.75M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z" />
			</svg>
			<p class="font-medium">No approved proofs yet</p>
			<p class="text-sm">Approved proofs will appear here for reference.</p>
		</div>
	{:else}
		<p class="text-sm text-gray-500 dark:text-gray-400 mt-4">{items.length} approved proof{items.length !== 1 ? 's' : ''}</p>

		{#each items as item (item.stage_id)}
			<div class="border dark:border-gray-700 rounded-xl overflow-hidden bg-white dark:bg-gray-900 shadow-sm mt-6">
				<!-- Header bar -->
				<div class="flex items-center gap-3 px-4 py-3 bg-gray-50 dark:bg-gray-800 border-b dark:border-gray-700">
					{#if item.item_type === 'new_proof'}
						<span class="px-2 py-0.5 text-xs font-semibold rounded bg-blue-100 text-blue-800 dark:bg-blue-900 dark:text-blue-200">NEW PROOF</span>
					{:else if item.item_type === 'reuse_check'}
						<span class="px-2 py-0.5 text-xs font-semibold rounded bg-amber-100 text-amber-800 dark:bg-amber-900 dark:text-amber-200">REUSE CHECK</span>
					{:else}
						<span class="px-2 py-0.5 text-xs font-semibold rounded bg-red-100 text-red-800 dark:bg-red-900 dark:text-red-200">UNPROVABLE</span>
					{/if}
					<span class="px-2 py-0.5 text-xs font-semibold rounded {statusBadgeClass(item.status)}">
						{item.status.replace('_', ' ').toUpperCase()}
					</span>
					<span class="text-xs text-gray-500 dark:text-gray-400">{formatDate(item.submitted_at)}</span>
					<span class="text-xs text-gray-400 dark:text-gray-500">Group: {item.group_id}</span>
				</div>

				<div class="p-4 flex flex-col gap-4">
					<!-- Original question -->
					<div>
						<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-1">Question</p>
						<p class="text-sm dark:text-white">{item.original_question}</p>
					</div>

					<!-- Agent reasoning -->
					<div>
						<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-1">Agent reasoning</p>
						<p class="text-sm text-gray-700 dark:text-gray-300 italic">{item.agent_reasoning}</p>
					</div>

					<!-- Decision notes -->
					{#if item.decision_notes}
						<div>
							<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-1">Review notes</p>
							<p class="text-sm text-gray-700 dark:text-gray-300">{item.decision_notes}</p>
						</div>
					{/if}

					<!-- Unprovable: ambiguity description -->
					{#if item.item_type === 'unprovable'}
						<div class="rounded-lg bg-red-50 dark:bg-red-950 border border-red-200 dark:border-red-800 p-3">
							<p class="text-xs font-semibold uppercase tracking-wide text-red-600 dark:text-red-400 mb-1">Policy ambiguity</p>
							<p class="text-sm text-red-800 dark:text-red-300">{item.unclear_aspect}</p>
						</div>
					{/if}

					<!-- Reuse check: imported modules -->
					{#if item.item_type === 'reuse_check' && item.reused_imports.length > 0}
						<div>
							<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-1">Reused library modules</p>
							<ul class="flex flex-col gap-1">
								{#each item.reused_imports as imp}
									<li class="text-sm font-mono text-amber-700 dark:text-amber-300 bg-amber-50 dark:bg-amber-950 rounded px-2 py-1">{imp}</li>
								{/each}
							</ul>
						</div>
					{/if}

					<!-- New proof: axioms, defs, theorem (read-only) -->
					{#if item.item_type === 'new_proof'}
						{#if item.new_axioms.length > 0}
							<div>
								<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-2">New axioms ({item.new_axioms.length})</p>
								<div class="flex flex-col gap-3">
									{#each item.new_axioms as axiom}
										<div class="rounded-lg border dark:border-gray-700 p-3 bg-gray-50 dark:bg-gray-800">
											<span class="font-mono text-sm font-semibold dark:text-white">{axiom.symbol}</span>
											{#if axiom.plain_comment}
												<p class="text-sm text-gray-600 dark:text-gray-300 mt-0.5">"{axiom.plain_comment}"</p>
											{/if}
											<pre class="text-xs font-mono bg-white dark:bg-gray-900 rounded p-2 border dark:border-gray-700 whitespace-pre-wrap overflow-x-auto mt-1">{axiom.lean_decl}</pre>
										</div>
									{/each}
								</div>
							</div>
						{/if}

						{#if item.new_defs.length > 0}
							<div>
								<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-2">Policy rules / defs ({item.new_defs.length})</p>
								<div class="flex flex-col gap-3">
									{#each item.new_defs as def}
										<div class="rounded-lg border dark:border-gray-700 p-3 bg-gray-50 dark:bg-gray-800">
											<span class="font-mono text-sm font-semibold dark:text-white">{def.symbol}</span>
											{#if def.plain_comment}
												<p class="text-sm text-gray-600 dark:text-gray-300 mt-0.5">"{def.plain_comment}"</p>
											{/if}
											<pre class="text-xs font-mono bg-white dark:bg-gray-900 rounded p-2 border dark:border-gray-700 whitespace-pre-wrap overflow-x-auto mt-1">{def.lean_decl}</pre>
										</div>
									{/each}
								</div>
							</div>
						{/if}

						{#if item.theorem}
							<div>
								<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-2">Theorem</p>
								<div class="rounded-lg border dark:border-gray-700 p-3 bg-gray-50 dark:bg-gray-800">
									<span class="font-mono text-sm font-semibold dark:text-white">{item.theorem.symbol}</span>
									{#if item.theorem.plain_comment}
										<p class="text-sm text-gray-600 dark:text-gray-300 mt-0.5">"{item.theorem.plain_comment}"</p>
									{/if}
									<pre class="text-xs font-mono bg-white dark:bg-gray-900 rounded p-2 border dark:border-gray-700 whitespace-pre-wrap overflow-x-auto mt-1">{item.theorem.lean_decl}</pre>
								</div>
							</div>
						{/if}
					{/if}
				</div>
			</div>
		{/each}
	{/if}
</div>
