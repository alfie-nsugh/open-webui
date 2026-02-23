<script lang="ts">
	import { onMount } from 'svelte';
	import { goto } from '$app/navigation';
	import { toast } from 'svelte-sonner';
	import { showPolicyCanvas, policyCanvasSessionId } from '$lib/stores';

	const API_BASE = `http://${window.location.hostname}:8000`;

	type PlanningSession = {
		session_id: string;
		group_id: string;
		question: string;
		status: string;
		created_at: string;
		assumption_count: number;
		resolved_count: number;
	};

	let planningSessions: PlanningSession[] = [];
	let planningLoading = true;

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

	// Per-item edit state: stage_id -> { [symbol]: edited_decl }
	let editState: Record<string, Record<string, string>> = {};
	// Per-item notes
	let notesState: Record<string, string> = {};
	// Per-item clarification text (for unprovable)
	let clarificationState: Record<string, string> = {};
	// Per-item inline edit toggle: stage_id -> { [symbol]: boolean }
	let editOpen: Record<string, Record<string, boolean>> = {};

	async function loadPlanningSessions() {
		planningLoading = true;
		try {
			const res = await fetch(`${API_BASE}/planning/active`);
			if (!res.ok) throw new Error(await res.text());
			planningSessions = await res.json();
		} catch (e: any) {
			toast.error(`Failed to load planning sessions: ${e.message}`);
		} finally {
			planningLoading = false;
		}
	}

	function resumeSession(session: PlanningSession) {
		policyCanvasSessionId.set(session.session_id);
		showPolicyCanvas.set(true);
		goto('/');
	}

	function statusBadgeClass(status: string): string {
		switch (status) {
			case 'discovery':
				return 'bg-purple-100 text-purple-800 dark:bg-purple-900 dark:text-purple-200';
			case 'planning':
				return 'bg-yellow-100 text-yellow-800 dark:bg-yellow-900 dark:text-yellow-200';
			case 'executing':
				return 'bg-blue-100 text-blue-800 dark:bg-blue-900 dark:text-blue-200';
			case 'post_flight':
				return 'bg-orange-100 text-orange-800 dark:bg-orange-900 dark:text-orange-200';
			case 'complete':
				return 'bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-200';
			default:
				return 'bg-gray-100 text-gray-800 dark:bg-gray-700 dark:text-gray-300';
		}
	}

	async function loadPending() {
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
			toast.error(`Failed to load review queue: ${e.message}`);
		} finally {
			loading = false;
			refreshing = false;
		}
	}

	onMount(() => {
		loadPending();
		loadPlanningSessions();
	});

	function getEditedLeanBody(item: ProofDetail): string | null {
		const edits = editState[item.stage_id];
		if (!edits || Object.keys(edits).length === 0) return null;

		if (!item.lean_body) return null;
		let body = item.lean_body;

		// Replace each edited declaration in the full lean_body
		for (const [_sym, newDecl] of Object.entries(edits)) {
			const orig = findOrigDecl(item, _sym);
			if (orig && newDecl !== orig) {
				body = body.replace(orig, newDecl);
			}
		}
		return body !== item.lean_body ? body : null;
	}

	function findOrigDecl(item: ProofDetail, symbol: string): string | null {
		const all = [...item.new_axioms, ...item.new_defs, ...(item.theorem ? [item.theorem] : [])];
		return all.find((a) => a.symbol === symbol)?.lean_decl ?? null;
	}

	async function decide(item: ProofDetail, decision: 'approve' | 'reject' | 'clarify', withEdits = false) {
		const edited_lean_body = withEdits ? getEditedLeanBody(item) : null;
		const payload: Record<string, any> = {
			decision,
			notes: notesState[item.stage_id] ?? ''
		};
		if (edited_lean_body) payload.edited_lean_body = edited_lean_body;
		if (decision === 'clarify') payload.clarification = clarificationState[item.stage_id] ?? '';

		try {
			const res = await fetch(`${API_BASE}/hitl/${item.stage_id}/decide`, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify(payload)
			});
			if (!res.ok) {
				const err = await res.json().catch(() => ({ detail: res.statusText }));
				const detail = typeof err.detail === 'object' ? JSON.stringify(err.detail) : err.detail;
				throw new Error(detail);
			}
			const result = await res.json();
			toast.success(`Decision recorded: ${result.status}`);
			await loadPending();
		} catch (e: any) {
			toast.error(`Decision failed: ${e.message}`);
		}
	}

	function toggleEdit(stageId: string, symbol: string, origDecl: string) {
		if (!editOpen[stageId]) editOpen[stageId] = {};
		editOpen[stageId][symbol] = !editOpen[stageId][symbol];
		// Pre-fill edit state with original if not already set
		if (!editState[stageId]) editState[stageId] = {};
		if (!editState[stageId][symbol]) editState[stageId][symbol] = origDecl;
		editOpen = editOpen; // trigger reactivity
	}

	function hasEdits(item: ProofDetail): boolean {
		return getEditedLeanBody(item) !== null;
	}

	function formatDate(iso: string): string {
		return new Date(iso).toLocaleString();
	}
</script>

<svelte:head>
	<title>Review Queue — BasedQED</title>
</svelte:head>

<div class="flex flex-col h-full w-full max-w-4xl mx-auto p-6 gap-6">
	<div class="flex items-center justify-between">
		<h1 class="text-2xl font-bold dark:text-white">Expert Review Queue</h1>
		<button
			class="flex items-center gap-2 px-3 py-1.5 text-sm rounded-lg border dark:border-gray-700 hover:bg-gray-100 dark:hover:bg-gray-800 transition"
			on:click={() => { loadPending(); loadPlanningSessions(); }}
			disabled={refreshing}
		>
			<svg xmlns="http://www.w3.org/2000/svg" class="size-4 {refreshing ? 'animate-spin' : ''}" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor">
				<path stroke-linecap="round" stroke-linejoin="round" d="M16.023 9.348h4.992v-.001M2.985 19.644v-4.992m0 0h4.992m-4.993 0 3.181 3.183a8.25 8.25 0 0 0 13.803-3.7M4.031 9.865a8.25 8.25 0 0 1 13.803-3.7l3.181 3.182m0-4.991v4.99" />
			</svg>
			Refresh
		</button>
	</div>

	<!-- Planning Sessions section -->
	<section>
		<h2 class="text-lg font-semibold dark:text-white mb-3">Planning Sessions</h2>
		{#if planningLoading}
			<p class="text-sm text-gray-500 dark:text-gray-400">Loading planning sessions...</p>
		{:else if planningSessions.length === 0}
			<div class="flex items-center gap-2 py-4 text-gray-500 dark:text-gray-400 text-sm">
				<svg xmlns="http://www.w3.org/2000/svg" class="size-5 opacity-40" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor">
					<path stroke-linecap="round" stroke-linejoin="round" d="M9 12h3.75M9 15h3.75M9 18h3.75m3 .75H18a2.25 2.25 0 0 0 2.25-2.25V6.108c0-1.135-.845-2.098-1.976-2.192a48.424 48.424 0 0 0-1.123-.08m-5.801 0c-.065.21-.1.433-.1.664 0 .414.336.75.75.75h4.5a.75.75 0 0 0 .75-.75 2.25 2.25 0 0 0-.1-.664m-5.8 0A2.251 2.251 0 0 1 13.5 2.25H15c1.012 0 1.867.668 2.15 1.586m-5.8 0c-.376.023-.75.05-1.124.08C9.095 4.01 8.25 4.973 8.25 6.108V8.25m0 0H4.875c-.621 0-1.125.504-1.125 1.125v11.25c0 .621.504 1.125 1.125 1.125h9.75c.621 0 1.125-.504 1.125-1.125V9.375c0-.621-.504-1.125-1.125-1.125H8.25Z" />
				</svg>
				No active planning sessions.
			</div>
		{:else}
			<div class="flex flex-col gap-2">
				{#each planningSessions as session (session.session_id)}
					<div class="border dark:border-gray-700 rounded-lg bg-white dark:bg-gray-900 shadow-sm p-4 flex items-start justify-between gap-4">
						<div class="flex-1 min-w-0">
							<p class="text-sm font-medium dark:text-white truncate">{session.question}</p>
							<div class="flex items-center gap-3 mt-1.5 flex-wrap">
								<span class="px-2 py-0.5 text-xs font-semibold rounded {statusBadgeClass(session.status)}">
									{session.status.replace('_', ' ').toUpperCase()}
								</span>
								<span class="text-xs text-gray-500 dark:text-gray-400">
									{session.resolved_count}/{session.assumption_count} assumptions resolved
								</span>
								<span class="text-xs text-gray-400 dark:text-gray-500">
									{formatDate(session.created_at)}
								</span>
							</div>
						</div>
						{#if session.status === 'post_flight'}
							<button
								class="px-3 py-1.5 text-sm font-medium rounded-lg bg-indigo-600 hover:bg-indigo-700 text-white transition shrink-0"
								on:click={() => resumeSession(session)}
							>
								Review
							</button>
						{:else if session.status === 'executing'}
							<span class="px-2 py-1 text-xs font-medium rounded bg-blue-100 dark:bg-blue-900 text-blue-700 dark:text-blue-300 shrink-0">
								Executing
							</span>
						{:else if session.status === 'planning'}
							<button
								class="px-3 py-1.5 text-sm font-medium rounded-lg bg-amber-600 hover:bg-amber-700 text-white transition shrink-0"
								on:click={() => resumeSession(session)}
							>
								Resolve
							</button>
						{:else}
							<span class="px-2 py-1 text-xs font-medium rounded bg-gray-100 dark:bg-gray-800 text-gray-500 dark:text-gray-400 shrink-0">
								{session.status}
							</span>
						{/if}
					</div>
				{/each}
			</div>
		{/if}
	</section>

	<!-- Proof Review Queue section -->
	<section>
		<h2 class="text-lg font-semibold dark:text-white mb-3">Proof Review Queue</h2>

	{#if loading}
		<div class="flex items-center justify-center py-20 text-gray-500 dark:text-gray-400">
			Loading review queue...
		</div>
	{:else if items.length === 0}
		<div class="flex flex-col items-center justify-center py-20 text-gray-500 dark:text-gray-400 gap-2">
			<svg xmlns="http://www.w3.org/2000/svg" class="size-12 opacity-40" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor">
				<path stroke-linecap="round" stroke-linejoin="round" d="M9 12.75 11.25 15 15 9.75M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z" />
			</svg>
			<p class="font-medium">No pending items</p>
			<p class="text-sm">All proofs and reuse checks have been reviewed.</p>
		</div>
	{:else}
		<p class="text-sm text-gray-500 dark:text-gray-400">{items.length} item{items.length !== 1 ? 's' : ''} pending review</p>

		{#each items as item (item.stage_id)}
			<div class="border dark:border-gray-700 rounded-xl overflow-hidden bg-white dark:bg-gray-900 shadow-sm">
				<!-- Header bar -->
				<div class="flex items-center gap-3 px-4 py-3 bg-gray-50 dark:bg-gray-800 border-b dark:border-gray-700">
					{#if item.item_type === 'new_proof'}
						<span class="px-2 py-0.5 text-xs font-semibold rounded bg-blue-100 text-blue-800 dark:bg-blue-900 dark:text-blue-200">NEW PROOF</span>
					{:else if item.item_type === 'reuse_check'}
						<span class="px-2 py-0.5 text-xs font-semibold rounded bg-amber-100 text-amber-800 dark:bg-amber-900 dark:text-amber-200">REUSE CHECK</span>
					{:else}
						<span class="px-2 py-0.5 text-xs font-semibold rounded bg-red-100 text-red-800 dark:bg-red-900 dark:text-red-200">UNPROVABLE</span>
					{/if}
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

					<!-- Unprovable: ambiguity description + clarification input -->
					{#if item.item_type === 'unprovable'}
						<div class="rounded-lg bg-red-50 dark:bg-red-950 border border-red-200 dark:border-red-800 p-3">
							<p class="text-xs font-semibold uppercase tracking-wide text-red-600 dark:text-red-400 mb-1">Policy ambiguity</p>
							<p class="text-sm text-red-800 dark:text-red-300">{item.unclear_aspect}</p>
						</div>
						<div>
							<label class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-1 block" for="clarify-{item.stage_id}">
								Provide clarification
							</label>
							<textarea
								id="clarify-{item.stage_id}"
								class="w-full text-sm rounded-lg border dark:border-gray-600 bg-white dark:bg-gray-800 dark:text-white p-2 min-h-[80px] resize-y focus:outline-none focus:ring-2 focus:ring-blue-500"
								placeholder="Enter the policy clarification or interpretation..."
								bind:value={clarificationState[item.stage_id]}
							></textarea>
						</div>
					{/if}

					<!-- PDC source text -->
					{#if item.provenance_text}
						<div>
							<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-1">Source PDC text</p>
							<div class="rounded-lg bg-gray-50 dark:bg-gray-800 border dark:border-gray-700 p-3 text-sm text-gray-700 dark:text-gray-300 whitespace-pre-wrap font-mono text-xs">
								{item.provenance_text}
							</div>
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
							<p class="text-xs text-gray-500 dark:text-gray-400 mt-1">
								↳ Verify these still apply to the <em>current</em> question (not just their original context).
							</p>
						</div>
					{/if}

					<!-- New proof: axioms, defs, theorem -->
					{#if item.item_type === 'new_proof'}
						{#if item.new_axioms.length > 0}
							<div>
								<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-2">New axioms ({item.new_axioms.length})</p>
								<div class="flex flex-col gap-3">
									{#each item.new_axioms as axiom}
										<div class="rounded-lg border dark:border-gray-700 p-3 bg-gray-50 dark:bg-gray-800">
											<div class="flex items-start justify-between gap-2 mb-1">
												<div>
													<span class="font-mono text-sm font-semibold dark:text-white">{axiom.symbol}</span>
													{#if axiom.plain_comment}
														<p class="text-sm text-gray-600 dark:text-gray-300 mt-0.5">"{axiom.plain_comment}"</p>
													{/if}
												</div>
												<button
													class="text-xs px-2 py-1 rounded border dark:border-gray-600 hover:bg-gray-200 dark:hover:bg-gray-700 transition shrink-0"
													on:click={() => toggleEdit(item.stage_id, axiom.symbol, axiom.lean_decl)}
												>
													{editOpen[item.stage_id]?.[axiom.symbol] ? 'Cancel' : '✏ Edit'}
												</button>
											</div>
											{#if editOpen[item.stage_id]?.[axiom.symbol]}
												<textarea
													class="w-full text-xs font-mono rounded border dark:border-gray-600 bg-white dark:bg-gray-900 dark:text-white p-2 min-h-[60px] resize-y focus:outline-none focus:ring-2 focus:ring-blue-500"
													bind:value={editState[item.stage_id][axiom.symbol]}
												></textarea>
											{:else}
												<pre class="text-xs font-mono bg-white dark:bg-gray-900 rounded p-2 border dark:border-gray-700 whitespace-pre-wrap overflow-x-auto">{axiom.lean_decl}</pre>
											{/if}
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
											<div class="flex items-start justify-between gap-2 mb-1">
												<div>
													<span class="font-mono text-sm font-semibold dark:text-white">{def.symbol}</span>
													{#if def.plain_comment}
														<p class="text-sm text-gray-600 dark:text-gray-300 mt-0.5">"{def.plain_comment}"</p>
													{/if}
												</div>
												<button
													class="text-xs px-2 py-1 rounded border dark:border-gray-600 hover:bg-gray-200 dark:hover:bg-gray-700 transition shrink-0"
													on:click={() => toggleEdit(item.stage_id, def.symbol, def.lean_decl)}
												>
													{editOpen[item.stage_id]?.[def.symbol] ? 'Cancel' : '✏ Edit'}
												</button>
											</div>
											{#if editOpen[item.stage_id]?.[def.symbol]}
												<textarea
													class="w-full text-xs font-mono rounded border dark:border-gray-600 bg-white dark:bg-gray-900 dark:text-white p-2 min-h-[80px] resize-y focus:outline-none focus:ring-2 focus:ring-blue-500"
													bind:value={editState[item.stage_id][def.symbol]}
												></textarea>
											{:else}
												<pre class="text-xs font-mono bg-white dark:bg-gray-900 rounded p-2 border dark:border-gray-700 whitespace-pre-wrap overflow-x-auto">{def.lean_decl}</pre>
											{/if}
										</div>
									{/each}
								</div>
							</div>
						{/if}

						{#if item.theorem}
							<div>
								<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-2">Theorem</p>
								<div class="rounded-lg border dark:border-gray-700 p-3 bg-gray-50 dark:bg-gray-800">
									<div class="flex items-start justify-between gap-2 mb-1">
										<div>
											<span class="font-mono text-sm font-semibold dark:text-white">{item.theorem.symbol}</span>
											{#if item.theorem.plain_comment}
												<p class="text-sm text-gray-600 dark:text-gray-300 mt-0.5">"{item.theorem.plain_comment}"</p>
											{/if}
										</div>
										<button
											class="text-xs px-2 py-1 rounded border dark:border-gray-600 hover:bg-gray-200 dark:hover:bg-gray-700 transition shrink-0"
											on:click={() => toggleEdit(item.stage_id, item.theorem!.symbol, item.theorem!.lean_decl)}
										>
											{editOpen[item.stage_id]?.[item.theorem.symbol] ? 'Cancel' : '✏ Edit'}
										</button>
									</div>
									{#if editOpen[item.stage_id]?.[item.theorem.symbol]}
										<textarea
											class="w-full text-xs font-mono rounded border dark:border-gray-600 bg-white dark:bg-gray-900 dark:text-white p-2 min-h-[100px] resize-y focus:outline-none focus:ring-2 focus:ring-blue-500"
											bind:value={editState[item.stage_id][item.theorem.symbol]}
										></textarea>
									{:else}
										<pre class="text-xs font-mono bg-white dark:bg-gray-900 rounded p-2 border dark:border-gray-700 whitespace-pre-wrap overflow-x-auto">{item.theorem.lean_decl}</pre>
									{/if}
								</div>
							</div>
						{/if}
					{/if}

					<!-- Notes field -->
					<div>
						<label class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-1 block" for="notes-{item.stage_id}">
							Notes (optional)
						</label>
						<input
							id="notes-{item.stage_id}"
							type="text"
							class="w-full text-sm rounded-lg border dark:border-gray-600 bg-white dark:bg-gray-800 dark:text-white px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
							placeholder="Add review notes..."
							bind:value={notesState[item.stage_id]}
						/>
					</div>

					<!-- Action buttons -->
					<div class="flex items-center gap-2 flex-wrap">
						{#if item.item_type === 'unprovable'}
							<button
								class="px-4 py-2 rounded-lg text-sm font-medium bg-blue-600 hover:bg-blue-700 text-white transition"
								on:click={() => decide(item, 'clarify')}
							>
								📝 Submit clarification
							</button>
							<button
								class="px-4 py-2 rounded-lg text-sm font-medium border dark:border-gray-600 hover:bg-gray-100 dark:hover:bg-gray-800 dark:text-white transition"
								on:click={() => decide(item, 'reject')}
							>
								✗ Mark unresolvable
							</button>
						{:else}
							<button
								class="px-4 py-2 rounded-lg text-sm font-medium bg-green-600 hover:bg-green-700 text-white transition"
								on:click={() => decide(item, 'approve', false)}
							>
								✓ Approve
							</button>
							{#if item.item_type === 'new_proof' && hasEdits(item)}
								<button
									class="px-4 py-2 rounded-lg text-sm font-medium bg-blue-600 hover:bg-blue-700 text-white transition"
									on:click={() => decide(item, 'approve', true)}
								>
									✎ Approve with edits
								</button>
							{/if}
							<button
								class="px-4 py-2 rounded-lg text-sm font-medium border dark:border-gray-600 hover:bg-red-50 dark:hover:bg-red-950 text-red-600 dark:text-red-400 border-red-200 dark:border-red-800 transition"
								on:click={() => decide(item, 'reject')}
							>
								✗ Reject
							</button>
						{/if}
					</div>
				</div>
			</div>
		{/each}
	{/if}
	</section>
</div>
