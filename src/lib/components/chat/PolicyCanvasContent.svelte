<script lang="ts">
	import { onDestroy, onMount } from 'svelte';
	import { showPolicyCanvas, policyCanvasSessionId } from '$lib/stores';

	// ── Props ──────────────────────────────────────────────────────────────
	export let sessionId = null;
	export let apiBase = 'http://localhost:8000';

	// ── Constants ──────────────────────────────────────────────────────────
	const POLL_INTERVAL_MS = 3000;

	// ── State ─────────────────────────────────────────────────────────────
	let session = null;
	let pollTimer = null;
	let error = null;

	// Per-assumption UI state keyed by assumption_id
	let selections = {};
	let customTexts = {};
	let submitting = {};

	// Resolved assumption collapse state (true = expanded)
	let expandedResolved = {};

	// Proof review state (replaces Confirm & Commit)
	let proofItem = null;
	let proofLoading = false;
	let proofEditState = {};
	let proofEditOpen = {};
	let proofNotes = '';
	let proofSubmitting = false;

	// ── Derived ───────────────────────────────────────────────────────────
	$: assumptions = session?.assumptions ?? [];
	$: resolvedCount = assumptions.filter((a) => a.status === 'resolved').length;
	$: totalCount = assumptions.length;
	$: allResolved = totalCount > 0 && resolvedCount === totalCount;
	$: progressPercent = totalCount > 0 ? (resolvedCount / totalCount) * 100 : 0;
	$: pendingAssumptions = assumptions.filter((a) => a.status === 'pending');
	$: resolvedAssumptions = assumptions.filter((a) => a.status === 'resolved');

	// Stop polling when session is done
	$: shouldPoll = session != null && session.status !== 'complete';

	// Auto-fetch proof details when post_flight has a stage_id
	$: if (
		session?.status === 'post_flight' &&
		session?.post_flight?.stage_id &&
		!proofItem &&
		!proofLoading
	) {
		fetchProofItem(session.post_flight.stage_id);
	}

	$: if (!shouldPoll && pollTimer) {
		clearInterval(pollTimer);
		pollTimer = null;
	}

	// ── API helpers ───────────────────────────────────────────────────────
	async function fetchSession() {
		const sid = sessionId ?? $policyCanvasSessionId;
		if (!sid) return;

		try {
			const res = await fetch(`${apiBase}/planning/${sid}`);
			if (!res.ok) {
				error = `Failed to fetch session: ${res.status}`;
				return;
			}
			session = await res.json();
			error = null;
		} catch (e) {
			error = `Connection error: ${e.message}`;
		}
	}

	async function decideAssumption(assumptionId) {
		const sid = sessionId ?? $policyCanvasSessionId;
		if (!sid) return;

		const selection = selections[assumptionId];
		if (!selection) return;

		const isCustom = selection === '__custom__';
		const customText = customTexts[assumptionId]?.trim();

		if (isCustom && !customText) return;

		submitting[assumptionId] = true;

		try {
			const body = {
				selected_option_id: isCustom ? null : selection,
				custom_response: isCustom ? customText : null
			};

			const res = await fetch(
				`${apiBase}/planning/${sid}/assumptions/${assumptionId}/decide`,
				{
					method: 'POST',
					headers: { 'Content-Type': 'application/json' },
					body: JSON.stringify(body)
				}
			);

			if (!res.ok) {
				if (res.status === 409) {
					error = 'This assumption has already been resolved.';
				} else if (res.status === 403) {
					error = 'Unable to submit — expert authentication required.';
				} else {
					const detail = await res.json().catch(() => null);
					error = detail?.detail || `Failed to submit decision: ${res.status}`;
				}
				return;
			}

			await fetchSession();
			error = null;

			// If all assumptions are now resolved, notify Chat.svelte to auto-continue
			const updated = session?.assumptions ?? [];
			const nowAllResolved =
				updated.length > 0 && updated.every((a) => a.status === 'resolved');
			if (nowAllResolved) {
				window.dispatchEvent(
					new CustomEvent('planning-resolved', {
						detail: { sessionId: sid }
					})
				);
				// Auto-close the canvas after a brief delay so user sees "all resolved"
				setTimeout(() => {
					showPolicyCanvas.set(false);
				}, 2000);
			}
		} catch (e) {
			error = `Connection error: ${e.message}`;
		} finally {
			submitting[assumptionId] = false;
		}
	}

	async function fetchProofItem(stageId) {
		proofLoading = true;
		try {
			const res = await fetch(`${apiBase}/hitl/${stageId}`);
			if (res.ok) {
				proofItem = await res.json();
			} else {
				error = `Failed to load proof: ${res.status}`;
			}
		} catch (e) {
			error = `Connection error: ${e.message}`;
		} finally {
			proofLoading = false;
		}
	}

	function toggleProofEdit(symbol, leanDecl) {
		if (!proofEditOpen) proofEditOpen = {};
		if (proofEditOpen[symbol]) {
			delete proofEditOpen[symbol];
			delete proofEditState[symbol];
		} else {
			proofEditOpen[symbol] = true;
			if (!proofEditState) proofEditState = {};
			proofEditState[symbol] = leanDecl;
		}
		proofEditOpen = proofEditOpen;
		proofEditState = proofEditState;
	}

	function hasProofEdits() {
		if (!proofItem || !proofEditState) return false;
		const allDecls = [
			...(proofItem.new_axioms || []),
			...(proofItem.new_defs || []),
			...(proofItem.theorem ? [proofItem.theorem] : [])
		];
		return allDecls.some((d) => {
			const edited = proofEditState[d.symbol];
			return edited !== undefined && edited !== d.lean_decl;
		});
	}

	function getEditedLeanBody() {
		if (!proofItem?.lean_body) return proofItem?.lean_body;
		let body = proofItem.lean_body;
		for (const [symbol, edited] of Object.entries(proofEditState || {})) {
			const allDecls = [
				...(proofItem.new_axioms || []),
				...(proofItem.new_defs || []),
				...(proofItem.theorem ? [proofItem.theorem] : [])
			];
			const original = allDecls.find((d) => d.symbol === symbol);
			if (original && edited !== original.lean_decl) {
				body = body.replace(original.lean_decl, edited);
			}
		}
		return body;
	}

	async function decideProof(decision, withEdits = false) {
		if (!proofItem) return;
		proofSubmitting = true;
		try {
			const payload = { decision, notes: proofNotes || '' };
			if (withEdits) {
				payload.edited_lean_body = getEditedLeanBody();
			}
			const res = await fetch(`${apiBase}/hitl/${proofItem.stage_id}/decide`, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify(payload)
			});
			if (!res.ok) {
				error = `Decision failed: ${res.status}`;
				return;
			}
			await fetchSession();
			proofItem = null;
			error = null;
		} catch (e) {
			error = `Connection error: ${e.message}`;
		} finally {
			proofSubmitting = false;
		}
	}

	// ── Polling lifecycle ─────────────────────────────────────────────────
	function startPolling() {
		if (pollTimer) clearInterval(pollTimer);
		pollTimer = setInterval(fetchSession, POLL_INTERVAL_MS);
	}

	function stopPolling() {
		if (pollTimer) {
			clearInterval(pollTimer);
			pollTimer = null;
		}
	}

	function closeCanvas() {
		showPolicyCanvas.set(false);
	}

	// ── Helper: find selected label for resolved assumption ───────────────
	function getResolvedLabel(assumption) {
		if (assumption.custom_response) {
			return `Custom: ${assumption.custom_response}`;
		}
		if (assumption.selected_option_id) {
			const opt = assumption.options.find(
				(o) => o.option_id === assumption.selected_option_id
			);
			return opt ? opt.label : assumption.selected_option_id;
		}
		return 'No selection';
	}

	// ── Lifecycle ─────────────────────────────────────────────────────────
	onMount(() => {
		fetchSession();
		startPolling();
	});

	onDestroy(() => {
		stopPolling();
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
			Policy Canvas
		</h2>
		<button
			class="p-1 rounded hover:bg-gray-100 dark:hover:bg-gray-800 text-gray-500 dark:text-gray-400 transition-colors"
			on:click={closeCanvas}
			aria-label="Close policy canvas"
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
		{#if !session}
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
				<p class="text-sm">Loading session...</p>
			</div>
		{:else if session.status === 'discovery'}
			<!-- Discovery state -->
			<div class="space-y-4">
				<div class="border-l-4 border-blue-500 dark:border-blue-400 pl-3">
					<p class="text-sm font-medium text-gray-900 dark:text-gray-100">
						{session.question}
					</p>
				</div>

				<div class="space-y-2 mt-6">
					<div class="flex items-center gap-2 text-sm text-gray-500 dark:text-gray-400">
						<svg
							class="animate-spin h-4 w-4"
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
						<span>Searching policy database...</span>
					</div>
					<div class="flex items-center gap-2 text-sm text-gray-500 dark:text-gray-400">
						<svg
							class="animate-spin h-4 w-4"
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
						<span>Identifying assumptions...</span>
					</div>
				</div>

				<p class="text-xs text-gray-400 dark:text-gray-500 mt-4">
					The agent is analyzing your question against the policy database. This takes a
					moment.
				</p>
			</div>
		{:else if session.status === 'planning'}
			<!-- Planning state (interactive) -->
			<div class="space-y-4">
				<div class="border-l-4 border-blue-500 dark:border-blue-400 pl-3">
					<p class="text-sm font-medium text-gray-900 dark:text-gray-100">
						{session.question}
					</p>
				</div>

				<!-- Progress bar -->
				<div class="space-y-1">
					<p class="text-xs text-gray-500 dark:text-gray-400 font-medium">
						{resolvedCount} of {totalCount} resolved
					</p>
					<div
						class="w-full h-2 bg-gray-200 dark:bg-gray-700 rounded-full overflow-hidden"
					>
						<div
							class="h-full bg-blue-500 dark:bg-blue-400 rounded-full transition-all duration-300"
							style="width: {progressPercent}%"
						/>
					</div>
				</div>

				<!-- Resolved assumptions (collapsible) -->
				{#each resolvedAssumptions as assumption (assumption.assumption_id)}
					<div
						class="border border-green-200 dark:border-green-800/50 rounded-lg overflow-hidden"
					>
						<button
							class="w-full flex items-center gap-2 px-3 py-2 text-left hover:bg-green-50 dark:hover:bg-green-900/20 transition-colors"
							on:click={() => {
								expandedResolved[assumption.assumption_id] =
									!expandedResolved[assumption.assumption_id];
							}}
						>
							<svg
								xmlns="http://www.w3.org/2000/svg"
								viewBox="0 0 20 20"
								fill="currentColor"
								class="w-4 h-4 text-green-600 dark:text-green-400 flex-shrink-0"
							>
								<path
									fill-rule="evenodd"
									d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z"
									clip-rule="evenodd"
								/>
							</svg>
							<span
								class="text-xs font-medium uppercase tracking-wide text-green-700 dark:text-green-400"
							>
								Resolved
							</span>
							<span class="text-sm text-gray-700 dark:text-gray-300 truncate flex-1">
								{assumption.question_text}
							</span>
							<svg
								xmlns="http://www.w3.org/2000/svg"
								viewBox="0 0 20 20"
								fill="currentColor"
								class="w-4 h-4 text-gray-400 transition-transform flex-shrink-0 {expandedResolved[
									assumption.assumption_id
								]
									? 'rotate-180'
									: ''}"
							>
								<path
									fill-rule="evenodd"
									d="M5.23 7.21a.75.75 0 011.06.02L10 11.168l3.71-3.938a.75.75 0 111.08 1.04l-4.25 4.5a.75.75 0 01-1.08 0l-4.25-4.5a.75.75 0 01.02-1.06z"
									clip-rule="evenodd"
								/>
							</svg>
						</button>
						{#if expandedResolved[assumption.assumption_id]}
							<div
								class="px-3 pb-3 border-t border-green-100 dark:border-green-800/30"
							>
								<p class="text-sm text-gray-600 dark:text-gray-400 mt-2">
									<span class="font-medium">Choice:</span>
									{getResolvedLabel(assumption)}
								</p>
								{#if assumption.source_reference}
									<p class="text-xs text-gray-400 dark:text-gray-500 mt-1">
										Source: {assumption.source_reference}
									</p>
								{/if}
							</div>
						{/if}
					</div>
				{/each}

				<!-- Pending assumptions -->
				{#each pendingAssumptions as assumption (assumption.assumption_id)}
					<div
						class="border border-amber-200 dark:border-amber-800/50 rounded-lg p-3 space-y-3"
					>
						<div class="flex items-start gap-2">
							<span
								class="text-amber-500 dark:text-amber-400 mt-0.5 flex-shrink-0 font-bold"
								>?</span
							>
							<div class="flex-1">
								<p
									class="text-xs font-medium uppercase tracking-wide text-amber-700 dark:text-amber-400"
								>
									Pending
								</p>
								<p
									class="text-sm font-medium text-gray-900 dark:text-gray-100 mt-0.5"
								>
									{assumption.question_text}
								</p>
							</div>
						</div>

						{#if assumption.source_reference}
							<p class="text-xs text-gray-400 dark:text-gray-500">
								Source: {assumption.source_reference}
							</p>
						{/if}
						{#if assumption.source_excerpt}
							<blockquote
								class="text-xs text-gray-600 dark:text-gray-400 border-l-2 border-gray-200 dark:border-gray-700 pl-2 italic"
							>
								{assumption.source_excerpt}
							</blockquote>
						{/if}
						{#if assumption.agent_reasoning}
							<p class="text-xs text-gray-500 dark:text-gray-400 italic">
								Agent: &ldquo;{assumption.agent_reasoning}&rdquo;
							</p>
						{/if}

						<!-- Radio options -->
						<div class="space-y-2">
							{#each assumption.options as option (option.option_id)}
								<label class="flex items-start gap-2 cursor-pointer group">
									<input
										type="radio"
										name="assumption-{assumption.assumption_id}"
										value={option.option_id}
										bind:group={selections[assumption.assumption_id]}
										class="mt-0.5 accent-blue-500"
									/>
									<div class="flex-1">
										<span
											class="text-sm text-gray-800 dark:text-gray-200 group-hover:text-gray-900 dark:group-hover:text-gray-100"
										>
											{option.label}
										</span>
										{#if option.is_recommended}
											<span
												class="ml-1 text-xs text-blue-600 dark:text-blue-400 font-medium"
												>(Rec.)</span
											>
										{/if}
										{#if option.description}
											<p class="text-xs text-gray-400 dark:text-gray-500 mt-0.5">
												{option.description}
											</p>
										{/if}
									</div>
								</label>
							{/each}

							<!-- Other / custom option -->
							<label class="flex items-start gap-2 cursor-pointer group">
								<input
									type="radio"
									name="assumption-{assumption.assumption_id}"
									value="__custom__"
									bind:group={selections[assumption.assumption_id]}
									class="mt-0.5 accent-blue-500"
								/>
								<span class="text-sm text-gray-800 dark:text-gray-200">Other:</span
								>
							</label>
							{#if selections[assumption.assumption_id] === '__custom__'}
								<input
									type="text"
									placeholder="Enter your interpretation..."
									bind:value={customTexts[assumption.assumption_id]}
									class="w-full text-sm px-3 py-1.5 rounded border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800 text-gray-900 dark:text-gray-100 placeholder-gray-400 dark:placeholder-gray-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:focus:ring-blue-400"
								/>
							{/if}
						</div>

						<!-- Confirm button -->
						<div class="flex justify-end">
							<button
								class="px-3 py-1.5 text-sm font-medium rounded bg-blue-600 hover:bg-blue-700 dark:bg-blue-500 dark:hover:bg-blue-600 text-white disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
								disabled={!selections[assumption.assumption_id] ||
									(selections[assumption.assumption_id] === '__custom__' &&
										!customTexts[assumption.assumption_id]?.trim()) ||
									submitting[assumption.assumption_id]}
								on:click={() => decideAssumption(assumption.assumption_id)}
							>
								{#if submitting[assumption.assumption_id]}
									Submitting...
								{:else}
									Confirm selection
								{/if}
							</button>
						</div>
					</div>
				{/each}

				</div>
		{:else if session.status === 'executing'}
			<!-- Executing state -->
			<div class="space-y-4">
				<div class="flex items-center gap-2">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						viewBox="0 0 20 20"
						fill="currentColor"
						class="w-5 h-5 text-green-600 dark:text-green-400"
					>
						<path
							fill-rule="evenodd"
							d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z"
							clip-rule="evenodd"
						/>
					</svg>
					<p class="text-sm text-gray-700 dark:text-gray-300">
						All {totalCount} assumptions established
					</p>
				</div>

				<div class="border-t border-gray-100 dark:border-gray-800 pt-3">
					<p
						class="text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-2"
					>
						Formalizing
					</p>
					<div class="space-y-2">
						<div
							class="flex items-center gap-2 text-sm text-gray-500 dark:text-gray-400"
						>
							<svg
								class="animate-spin h-4 w-4"
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
							<span>Constructing proof...</span>
						</div>
						<div
							class="flex items-center gap-2 text-sm text-gray-400 dark:text-gray-500"
						>
							<div class="w-4 h-4 flex items-center justify-center">
								<div
									class="w-1.5 h-1.5 rounded-full bg-gray-300 dark:bg-gray-600"
								/>
							</div>
							<span>Validating...</span>
						</div>
					</div>
				</div>
			</div>
		{:else if session.status === 'post_flight'}
			<div class="space-y-4">
				{#if session.post_flight}
					<div class="flex items-center gap-2">
						<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5 text-green-600 dark:text-green-400">
							<path fill-rule="evenodd" d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z" clip-rule="evenodd" />
						</svg>
						<p class="text-sm font-medium text-gray-700 dark:text-gray-300">Formalization complete — review proof</p>
					</div>
					<p class="text-sm text-gray-600 dark:text-gray-400">{session.post_flight.summary}</p>

					{#if proofLoading}
						<p class="text-sm text-gray-400">Loading proof details...</p>
					{:else if proofItem}
						<!-- Proof review UI -->
						{#if proofItem.item_type === 'new_proof'}
							{#if proofItem.new_axioms?.length > 0}
								<div>
									<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-2">New axioms ({proofItem.new_axioms.length})</p>
									<div class="flex flex-col gap-2">
										{#each proofItem.new_axioms as axiom}
											<div class="rounded-lg border dark:border-gray-700 p-3 bg-gray-50 dark:bg-gray-800">
												<div class="flex items-start justify-between gap-2 mb-1">
													<div>
														<span class="font-mono text-sm font-semibold dark:text-white">{axiom.symbol}</span>
														{#if axiom.plain_comment}
															<p class="text-sm text-gray-600 dark:text-gray-300 mt-0.5">"{axiom.plain_comment}"</p>
														{/if}
													</div>
													<button class="text-xs px-2 py-1 rounded border dark:border-gray-600 hover:bg-gray-200 dark:hover:bg-gray-700 transition shrink-0" on:click={() => toggleProofEdit(axiom.symbol, axiom.lean_decl)}>
														{proofEditOpen[axiom.symbol] ? 'Cancel' : 'Edit'}
													</button>
												</div>
												{#if proofEditOpen[axiom.symbol]}
													<textarea class="w-full text-xs font-mono rounded border dark:border-gray-600 bg-white dark:bg-gray-900 dark:text-white p-2 min-h-[60px] resize-y focus:outline-none focus:ring-2 focus:ring-blue-500" bind:value={proofEditState[axiom.symbol]}></textarea>
												{:else}
													<pre class="text-xs font-mono bg-white dark:bg-gray-900 rounded p-2 border dark:border-gray-700 whitespace-pre-wrap overflow-x-auto">{axiom.lean_decl}</pre>
												{/if}
											</div>
										{/each}
									</div>
								</div>
							{/if}

							{#if proofItem.new_defs?.length > 0}
								<div>
									<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-2">Policy rules / defs ({proofItem.new_defs.length})</p>
									<div class="flex flex-col gap-2">
										{#each proofItem.new_defs as def}
											<div class="rounded-lg border dark:border-gray-700 p-3 bg-gray-50 dark:bg-gray-800">
												<div class="flex items-start justify-between gap-2 mb-1">
													<div>
														<span class="font-mono text-sm font-semibold dark:text-white">{def.symbol}</span>
														{#if def.plain_comment}
															<p class="text-sm text-gray-600 dark:text-gray-300 mt-0.5">"{def.plain_comment}"</p>
														{/if}
													</div>
													<button class="text-xs px-2 py-1 rounded border dark:border-gray-600 hover:bg-gray-200 dark:hover:bg-gray-700 transition shrink-0" on:click={() => toggleProofEdit(def.symbol, def.lean_decl)}>
														{proofEditOpen[def.symbol] ? 'Cancel' : 'Edit'}
													</button>
												</div>
												{#if proofEditOpen[def.symbol]}
													<textarea class="w-full text-xs font-mono rounded border dark:border-gray-600 bg-white dark:bg-gray-900 dark:text-white p-2 min-h-[80px] resize-y focus:outline-none focus:ring-2 focus:ring-blue-500" bind:value={proofEditState[def.symbol]}></textarea>
												{:else}
													<pre class="text-xs font-mono bg-white dark:bg-gray-900 rounded p-2 border dark:border-gray-700 whitespace-pre-wrap overflow-x-auto">{def.lean_decl}</pre>
												{/if}
											</div>
										{/each}
									</div>
								</div>
							{/if}

							{#if proofItem.theorem}
								<div>
									<p class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-2">Theorem</p>
									<div class="rounded-lg border dark:border-gray-700 p-3 bg-gray-50 dark:bg-gray-800">
										<div class="flex items-start justify-between gap-2 mb-1">
											<div>
												<span class="font-mono text-sm font-semibold dark:text-white">{proofItem.theorem.symbol}</span>
												{#if proofItem.theorem.plain_comment}
													<p class="text-sm text-gray-600 dark:text-gray-300 mt-0.5">"{proofItem.theorem.plain_comment}"</p>
												{/if}
											</div>
											<button class="text-xs px-2 py-1 rounded border dark:border-gray-600 hover:bg-gray-200 dark:hover:bg-gray-700 transition shrink-0" on:click={() => toggleProofEdit(proofItem.theorem.symbol, proofItem.theorem.lean_decl)}>
												{proofEditOpen[proofItem.theorem.symbol] ? 'Cancel' : 'Edit'}
											</button>
										</div>
										{#if proofEditOpen[proofItem.theorem.symbol]}
											<textarea class="w-full text-xs font-mono rounded border dark:border-gray-600 bg-white dark:bg-gray-900 dark:text-white p-2 min-h-[100px] resize-y focus:outline-none focus:ring-2 focus:ring-blue-500" bind:value={proofEditState[proofItem.theorem.symbol]}></textarea>
										{:else}
											<pre class="text-xs font-mono bg-white dark:bg-gray-900 rounded p-2 border dark:border-gray-700 whitespace-pre-wrap overflow-x-auto">{proofItem.theorem.lean_decl}</pre>
										{/if}
									</div>
								</div>
							{/if}
						{/if}

						<!-- Notes -->
						<div>
							<label class="text-xs font-semibold uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-1 block">Notes (optional)</label>
							<input type="text" class="w-full text-sm rounded-lg border dark:border-gray-600 bg-white dark:bg-gray-800 dark:text-white px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500" placeholder="Add review notes..." bind:value={proofNotes} />
						</div>

						<!-- Action buttons -->
						<div class="flex items-center gap-2 flex-wrap pt-2">
							<button class="flex-1 py-2 text-sm font-medium rounded-lg bg-green-600 hover:bg-green-700 text-white disabled:opacity-50 transition-colors" disabled={proofSubmitting} on:click={() => decideProof('approve')}>
								{proofSubmitting ? 'Approving...' : 'Approve'}
							</button>
							{#if hasProofEdits()}
								<button class="flex-1 py-2 text-sm font-medium rounded-lg bg-blue-600 hover:bg-blue-700 text-white disabled:opacity-50 transition-colors" disabled={proofSubmitting} on:click={() => decideProof('approve', true)}>
									Approve with edits
								</button>
							{/if}
							<button class="px-3 py-2 text-sm font-medium rounded-lg border border-red-200 dark:border-red-800 text-red-600 dark:text-red-400 hover:bg-red-50 dark:hover:bg-red-950 disabled:opacity-50 transition-colors" disabled={proofSubmitting} on:click={() => decideProof('reject')}>
								Reject
							</button>
						</div>
					{/if}
				{:else}
					<p class="text-sm text-gray-500 dark:text-gray-400">Waiting for post-flight summary from agent...</p>
				{/if}
			</div>
		{:else if session.status === 'complete'}
			<!-- Complete state -->
			<div class="space-y-4">
				<div class="flex items-center gap-2">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						viewBox="0 0 20 20"
						fill="currentColor"
						class="w-5 h-5 text-green-600 dark:text-green-400"
					>
						<path
							fill-rule="evenodd"
							d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z"
							clip-rule="evenodd"
						/>
					</svg>
					<p class="text-sm font-medium text-gray-700 dark:text-gray-300">
						Session complete
					</p>
				</div>

				{#if session.post_flight}
					<p class="text-sm text-gray-600 dark:text-gray-400">
						{session.post_flight.summary}
					</p>
				{/if}

				{#if resolvedAssumptions.length > 0}
					<div class="border-t border-gray-100 dark:border-gray-800 pt-3">
						<p
							class="text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-gray-400 mb-2"
						>
							Established assumptions ({resolvedAssumptions.length})
						</p>
						{#each resolvedAssumptions as assumption (assumption.assumption_id)}
							<div class="py-1.5 text-sm text-gray-600 dark:text-gray-400">
								<span class="text-green-600 dark:text-green-400 mr-1"
									>&#10003;</span
								>
								{assumption.question_text} &mdash; {getResolvedLabel(assumption)}
							</div>
						{/each}
					</div>
				{/if}
			</div>
		{/if}
	</div>
</div>
