<script lang="ts">
	import { onMount } from 'svelte';
	import { showPolicyCanvas } from '$lib/stores';

	// ── Props ──────────────────────────────────────────────────────────────
	export let batchId: string;
	export let apiBase: string = 'http://localhost:8000';

	// ── Types ─────────────────────────────────────────────────────────────
	type DefinitionItem = {
		filename: string;
		cnl: string;
		source_snippet: string;
		decision: 'approved' | 'rejected' | null;
		rejection_reason: string | null;
	};

	type BatchData = {
		batch_id: string;
		conversation_id: string;
		group_id: string;
		snapshot_hash: string;
		phase: 'reviewing' | 'complete' | 'rejected';
		created_at: string;
		original_question: string | null;
		definitions: DefinitionItem[];
		lemmas: string[];
		question_filename: string | null;
		question_cnl: string | null;
		question_decision: 'approved' | 'rejected' | null;
		question_rejection_reason: string | null;
	};

	// ── State ─────────────────────────────────────────────────────────────
	let batch: BatchData | null = null;
	let loading = true;
	let error: string | null = null;
	let rejectionReasons: Record<string, string> = {};
	let questionRejectionReason = '';
	let submitting: Record<string, boolean> = {};
	let rejectingFilename: string | null = null; // which definition has reject input open
	let rejectingQuestion = false; // whether question reject input is open

	// ── Derived ───────────────────────────────────────────────────────────
	$: definitions = batch?.definitions ?? [];
	$: reviewedCount = definitions.filter((d) => d.decision !== null).length +
		(batch?.question_filename && batch?.question_decision !== null ? 1 : 0);
	$: totalCount = definitions.length +
		(batch?.question_filename ? 1 : 0);
	$: isComplete = batch?.phase === 'complete';
	$: isRejected = batch?.phase === 'rejected';

	// ── API helpers ───────────────────────────────────────────────────────
	async function refreshBatch() {
		try {
			const res = await fetch(`${apiBase}/hitl/batch/${batchId}`);
			if (!res.ok) {
				error = `Failed to fetch batch: ${res.status}`;
				return;
			}
			batch = await res.json();
			error = null;
		} catch (e: any) {
			error = `Connection error: ${e.message}`;
		}
	}

	async function approveDefinition(filename: string) {
		submitting[filename] = true;
		try {
			const res = await fetch(
				`${apiBase}/hitl/batch/${batchId}/definitions/${filename}/decide`,
				{
					method: 'POST',
					headers: { 'Content-Type': 'application/json' },
					body: JSON.stringify({ decision: 'approved' }),
				}
			);
			if (res.ok) {
				await refreshBatch();
				checkForCompletion();
			} else {
				error = `Failed to approve: ${res.status}`;
			}
		} catch (e: any) {
			error = `Connection error: ${e.message}`;
		} finally {
			submitting[filename] = false;
		}
	}

	async function rejectDefinition(filename: string) {
		submitting[filename] = true;
		const reason = rejectionReasons[filename] || '';
		try {
			const res = await fetch(
				`${apiBase}/hitl/batch/${batchId}/definitions/${filename}/decide`,
				{
					method: 'POST',
					headers: { 'Content-Type': 'application/json' },
					body: JSON.stringify({ decision: 'rejected', rejection_reason: reason }),
				}
			);
			if (res.ok) {
				rejectingFilename = null;
				await refreshBatch();
				checkForRejection();
			} else {
				error = `Failed to reject: ${res.status}`;
			}
		} catch (e: any) {
			error = `Connection error: ${e.message}`;
		} finally {
			submitting[filename] = false;
		}
	}

	async function approveQuestion() {
		submitting['__question__'] = true;
		try {
			const res = await fetch(
				`${apiBase}/hitl/batch/${batchId}/question/decide`,
				{
					method: 'POST',
					headers: { 'Content-Type': 'application/json' },
					body: JSON.stringify({ decision: 'approved' }),
				}
			);
			if (res.ok) {
				await refreshBatch();
				checkForCompletion();
			} else {
				error = `Failed to approve question: ${res.status}`;
			}
		} catch (e: any) {
			error = `Connection error: ${e.message}`;
		} finally {
			submitting['__question__'] = false;
		}
	}

	async function rejectQuestion() {
		submitting['__question__'] = true;
		try {
			const res = await fetch(
				`${apiBase}/hitl/batch/${batchId}/question/decide`,
				{
					method: 'POST',
					headers: { 'Content-Type': 'application/json' },
					body: JSON.stringify({ decision: 'rejected', rejection_reason: questionRejectionReason }),
				}
			);
			if (res.ok) {
				rejectingQuestion = false;
				await refreshBatch();
				checkForRejection();
			} else {
				error = `Failed to reject question: ${res.status}`;
			}
		} catch (e: any) {
			error = `Connection error: ${e.message}`;
		} finally {
			submitting['__question__'] = false;
		}
	}

	function checkForRejection() {
		if (batch?.phase === 'rejected') {
			const rejectedDef = batch.definitions.find((d) => d.decision === 'rejected');
			const detail = {
				batchId: batch.batch_id,
				groupId: batch.group_id,
				question: batch.original_question || '',
				decision: 'reject',
				notes: rejectedDef?.rejection_reason || batch.question_rejection_reason || '',
				filename: rejectedDef?.filename || batch.question_filename || '',
			};
			window.dispatchEvent(new CustomEvent('hitl-feedback', { detail, bubbles: true }));
			setTimeout(() => {
				showPolicyCanvas.set(false);
			}, 2000);
		}
	}

	function checkForCompletion() {
		if (batch?.phase === 'complete') {
			// Success — canvas stays open briefly showing the complete state
			setTimeout(() => {
				showPolicyCanvas.set(false);
			}, 2000);
		}
	}

	function closeCanvas() {
		showPolicyCanvas.set(false);
	}

	// ── Lifecycle ─────────────────────────────────────────────────────────
	onMount(async () => {
		await refreshBatch();
		loading = false;
	});
</script>

<div class="flex flex-col h-full">
	<!-- Header -->
	<div
		class="flex items-center justify-between px-4 py-3 border-b border-gray-100 dark:border-gray-800"
	>
		<h2
			class="text-xs font-semibold tracking-widest uppercase text-gray-500 dark:text-gray-400"
		>
			Review
		</h2>
		<button
			class="p-1 rounded hover:bg-gray-100 dark:hover:bg-gray-800 text-gray-500 dark:text-gray-400 transition-colors"
			on:click={closeCanvas}
			aria-label="Close review canvas"
		>
			<svg
				xmlns="http://www.w3.org/2000/svg"
				viewBox="0 0 20 20"
				fill="currentColor"
				class="w-4 h-4"
			>
				<path
					d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z"
				/>
			</svg>
		</button>
	</div>

	<!-- Error banner -->
	{#if error}
		<div
			class="px-4 py-2 text-sm bg-red-50 dark:bg-red-900/20 text-red-700 dark:text-red-300 border-b border-red-100 dark:border-red-800"
		>
			{error}
		</div>
	{/if}

	<!-- Body -->
	<div class="flex-1 overflow-y-auto scrollbar-hidden px-4 py-3">
		{#if loading}
			<!-- Loading state -->
			<div
				class="flex flex-col items-center justify-center h-full text-gray-400 dark:text-gray-500 gap-3"
			>
				<svg
					class="animate-spin h-6 w-6"
					xmlns="http://www.w3.org/2000/svg"
					fill="none"
					viewBox="0 0 24 24"
				>
					<circle
						class="opacity-25"
						cx="12"
						cy="12"
						r="10"
						stroke="currentColor"
						stroke-width="4"
					/>
					<path
						class="opacity-75"
						fill="currentColor"
						d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
					/>
				</svg>
				<p class="text-sm">Loading batch...</p>
			</div>
		{:else if !batch}
			<div class="flex flex-col items-center justify-center h-full text-gray-400 dark:text-gray-500 gap-2">
				<p class="text-sm">Batch not found.</p>
			</div>
		{:else if isComplete}
			<!-- Complete state -->
			<div class="flex flex-col items-center justify-center h-full gap-3">
				<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-10 h-10 text-green-500">
					<path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.857-9.809a.75.75 0 00-1.214-.882l-3.483 4.79-1.88-1.88a.75.75 0 10-1.06 1.061l2.5 2.5a.75.75 0 001.137-.089l4-5.5z" clip-rule="evenodd" />
				</svg>
				<p class="text-sm font-medium text-green-700 dark:text-green-400">All items approved — batch complete</p>
			</div>
		{:else if isRejected}
			<!-- Rejected state -->
			<div class="flex flex-col items-center justify-center h-full gap-3">
				<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-10 h-10 text-red-500">
					<path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.28 7.22a.75.75 0 00-1.06 1.06L8.94 10l-1.72 1.72a.75.75 0 101.06 1.06L10 11.06l1.72 1.72a.75.75 0 101.06-1.06L11.06 10l1.72-1.72a.75.75 0 00-1.06-1.06L10 8.94 8.28 7.22z" clip-rule="evenodd" />
				</svg>
				<p class="text-sm font-medium text-red-700 dark:text-red-400">Batch rejected — returning to agent</p>
			</div>
		{:else}
			<!-- Reviewing state -->
			<div class="space-y-6">
				<!-- Original question -->
				{#if batch.original_question}
					<div class="border-l-4 border-blue-500 dark:border-blue-400 pl-3">
						<p class="text-sm font-medium text-gray-900 dark:text-gray-100">
							{batch.original_question}
						</p>
					</div>
				{/if}

				<!-- Definitions & Constraints section -->
				{#if definitions.length > 0}
					<div class="space-y-3">
						<h3 class="text-xs font-semibold tracking-widest uppercase text-gray-500 dark:text-gray-400">
							Definitions & Constraints
						</h3>

						{#each definitions as def (def.filename)}
							<div class="rounded-lg border dark:border-gray-700 p-4 bg-white dark:bg-gray-900 space-y-3">
								<!-- CNL text -->
								<pre class="text-sm text-gray-900 dark:text-gray-100 whitespace-pre-wrap font-mono leading-relaxed">{def.cnl}</pre>

								<!-- Source snippet -->
								{#if def.source_snippet}
									<div class="rounded bg-gray-50 dark:bg-gray-800 px-3 py-2">
										<p class="text-xs text-gray-500 dark:text-gray-400 mb-1 font-medium">Source</p>
										<p class="text-xs text-gray-600 dark:text-gray-300 italic">{def.source_snippet}</p>
									</div>
								{/if}

								<!-- Decision display / actions -->
								{#if def.decision === 'approved'}
									<div class="flex items-center gap-2">
										<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5 text-green-500">
											<path fill-rule="evenodd" d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z" clip-rule="evenodd" />
										</svg>
										<span class="text-sm font-medium text-green-700 dark:text-green-400">Approved</span>
									</div>
								{:else if def.decision === 'rejected'}
									<div class="flex items-center gap-2">
										<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5 text-red-500">
											<path d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z" />
										</svg>
										<span class="text-sm font-medium text-red-700 dark:text-red-400">Rejected</span>
										{#if def.rejection_reason}
											<span class="text-xs text-gray-500 dark:text-gray-400">— {def.rejection_reason}</span>
										{/if}
									</div>
								{:else}
									<!-- Pending: show action buttons -->
									<div class="flex items-center gap-2">
										<button
											class="px-3 py-1.5 text-xs font-medium rounded-md bg-green-600 hover:bg-green-700 text-white transition-colors disabled:opacity-50"
											disabled={submitting[def.filename]}
											on:click={() => approveDefinition(def.filename)}
										>
											{submitting[def.filename] ? 'Approving...' : 'Approve'}
										</button>
										{#if rejectingFilename === def.filename}
											<div class="flex-1 flex items-center gap-2">
												<input
													type="text"
													placeholder="Reason (optional)"
													bind:value={rejectionReasons[def.filename]}
													class="flex-1 text-xs px-2 py-1.5 rounded-md border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 text-gray-900 dark:text-gray-100 focus:outline-none focus:ring-1 focus:ring-red-500"
													on:keydown={(e) => { if (e.key === 'Enter') rejectDefinition(def.filename); }}
												/>
												<button
													class="px-3 py-1.5 text-xs font-medium rounded-md bg-red-600 hover:bg-red-700 text-white transition-colors disabled:opacity-50"
													disabled={submitting[def.filename]}
													on:click={() => rejectDefinition(def.filename)}
												>
													{submitting[def.filename] ? 'Rejecting...' : 'Confirm'}
												</button>
												<button
													class="px-2 py-1.5 text-xs text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200"
													on:click={() => { rejectingFilename = null; }}
												>
													Cancel
												</button>
											</div>
										{:else}
											<button
												class="px-3 py-1.5 text-xs font-medium rounded-md border border-red-300 dark:border-red-700 text-red-700 dark:text-red-400 hover:bg-red-50 dark:hover:bg-red-900/20 transition-colors"
												on:click={() => { rejectingFilename = def.filename; }}
											>
												Reject
											</button>
										{/if}
									</div>
								{/if}

								<!-- Filename label -->
								<p class="text-xs text-gray-400 dark:text-gray-500 font-mono">{def.filename}</p>
							</div>
						{/each}
					</div>
				{/if}

				<!-- Question Theorem section -->
				{#if batch.question_filename}
					<div class="space-y-3">
						<h3 class="text-xs font-semibold tracking-widest uppercase text-gray-500 dark:text-gray-400">
							Question Theorem
						</h3>

						<div class="rounded-lg border dark:border-gray-700 p-4 bg-white dark:bg-gray-900 space-y-3">
							<!-- Question CNL -->
							{#if batch.question_cnl}
								<pre class="text-sm text-gray-900 dark:text-gray-100 whitespace-pre-wrap font-mono leading-relaxed">{batch.question_cnl}</pre>
							{/if}

							<!-- Original question -->
							{#if batch.original_question}
								<div class="rounded bg-gray-50 dark:bg-gray-800 px-3 py-2">
									<p class="text-xs text-gray-500 dark:text-gray-400 mb-1 font-medium">Original question</p>
									<p class="text-xs text-gray-600 dark:text-gray-300 italic">{batch.original_question}</p>
								</div>
							{/if}

							<!-- Decision display / actions -->
							{#if batch.question_decision === 'approved'}
								<div class="flex items-center gap-2">
									<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5 text-green-500">
										<path fill-rule="evenodd" d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z" clip-rule="evenodd" />
									</svg>
									<span class="text-sm font-medium text-green-700 dark:text-green-400">Approved</span>
								</div>
							{:else if batch.question_decision === 'rejected'}
								<div class="flex items-center gap-2">
									<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5 text-red-500">
										<path d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z" />
									</svg>
									<span class="text-sm font-medium text-red-700 dark:text-red-400">Rejected</span>
									{#if batch.question_rejection_reason}
										<span class="text-xs text-gray-500 dark:text-gray-400">— {batch.question_rejection_reason}</span>
									{/if}
								</div>
							{:else}
								<!-- Pending: show action buttons -->
								<div class="flex items-center gap-2">
									<button
										class="px-3 py-1.5 text-xs font-medium rounded-md bg-green-600 hover:bg-green-700 text-white transition-colors disabled:opacity-50"
										disabled={submitting['__question__']}
										on:click={approveQuestion}
									>
										{submitting['__question__'] ? 'Approving...' : 'Approve'}
									</button>
									{#if rejectingQuestion}
										<div class="flex-1 flex items-center gap-2">
											<input
												type="text"
												placeholder="Reason (optional)"
												bind:value={questionRejectionReason}
												class="flex-1 text-xs px-2 py-1.5 rounded-md border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 text-gray-900 dark:text-gray-100 focus:outline-none focus:ring-1 focus:ring-red-500"
												on:keydown={(e) => { if (e.key === 'Enter') rejectQuestion(); }}
											/>
											<button
												class="px-3 py-1.5 text-xs font-medium rounded-md bg-red-600 hover:bg-red-700 text-white transition-colors disabled:opacity-50"
												disabled={submitting['__question__']}
												on:click={rejectQuestion}
											>
												{submitting['__question__'] ? 'Rejecting...' : 'Confirm'}
											</button>
											<button
												class="px-2 py-1.5 text-xs text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200"
												on:click={() => { rejectingQuestion = false; }}
											>
												Cancel
											</button>
										</div>
									{:else}
										<button
											class="px-3 py-1.5 text-xs font-medium rounded-md border border-red-300 dark:border-red-700 text-red-700 dark:text-red-400 hover:bg-red-50 dark:hover:bg-red-900/20 transition-colors"
											on:click={() => { rejectingQuestion = true; }}
										>
											Reject
										</button>
									{/if}
								</div>
							{/if}

							<!-- Filename label -->
							<p class="text-xs text-gray-400 dark:text-gray-500 font-mono">{batch.question_filename}</p>
						</div>
					</div>
				{/if}

				<!-- Lemmas section -->
				{#if batch.lemmas && batch.lemmas.length > 0}
					<div class="space-y-2">
						<h3 class="text-xs font-semibold tracking-widest uppercase text-gray-500 dark:text-gray-400">
							Lemmas (auto-approved)
						</h3>

						{#each batch.lemmas as lemma}
							<div class="flex items-center gap-2 px-3 py-2 rounded-lg bg-gray-50 dark:bg-gray-800">
								<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-4 h-4 text-green-500 flex-shrink-0">
									<path fill-rule="evenodd" d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z" clip-rule="evenodd" />
								</svg>
								<span class="text-xs text-gray-700 dark:text-gray-300 font-mono">{lemma}</span>
							</div>
						{/each}
					</div>
				{/if}
			</div>
		{/if}
	</div>

	<!-- Footer: progress bar -->
	{#if batch && !isComplete && !isRejected}
		<div class="px-4 py-3 border-t border-gray-100 dark:border-gray-800">
			<div class="flex items-center justify-between mb-1">
				<p class="text-xs text-gray-500 dark:text-gray-400 font-medium">
					{reviewedCount} of {totalCount} reviewed
				</p>
				{#if batch.phase === 'reviewing'}
					<span class="text-xs text-amber-600 dark:text-amber-400 font-medium">Reviewing</span>
				{/if}
			</div>
			<div class="w-full h-1.5 bg-gray-200 dark:bg-gray-700 rounded-full overflow-hidden">
				<div
					class="h-full bg-blue-500 dark:bg-blue-400 rounded-full transition-all duration-300"
					style="width: {totalCount > 0 ? (reviewedCount / totalCount) * 100 : 0}%"
				/>
			</div>
		</div>
	{/if}
</div>
