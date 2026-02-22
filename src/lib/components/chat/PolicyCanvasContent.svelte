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

	// Post-flight state
	let confirmSubmitting = false;
	let flagSubmitting = false;
	let flagReason = '';
	let showFlagInput = false;

	// ── Derived ───────────────────────────────────────────────────────────
	$: assumptions = session?.assumptions ?? [];
	$: resolvedCount = assumptions.filter((a) => a.status === 'resolved').length;
	$: totalCount = assumptions.length;
	$: allResolved = totalCount > 0 && resolvedCount === totalCount;
	$: progressPercent = totalCount > 0 ? (resolvedCount / totalCount) * 100 : 0;
	$: pendingAssumptions = assumptions.filter((a) => a.status === 'pending');
	$: resolvedAssumptions = assumptions.filter((a) => a.status === 'resolved');

	// Stop polling when session is done
	$: shouldPoll =
		session != null &&
		session.status !== 'complete' &&
		!(session.status === 'post_flight' && session.post_flight?.status === 'confirmed');

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
				error = `Failed to submit decision: ${res.status}`;
				return;
			}

			await fetchSession();
			error = null;
		} catch (e) {
			error = `Connection error: ${e.message}`;
		} finally {
			submitting[assumptionId] = false;
		}
	}

	async function confirmPostFlight() {
		const sid = sessionId ?? $policyCanvasSessionId;
		if (!sid) return;

		confirmSubmitting = true;
		try {
			const res = await fetch(`${apiBase}/planning/${sid}/confirm`, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ confirm: true })
			});

			if (!res.ok) {
				error = `Failed to confirm: ${res.status}`;
				return;
			}

			await fetchSession();
			error = null;
		} catch (e) {
			error = `Connection error: ${e.message}`;
		} finally {
			confirmSubmitting = false;
		}
	}

	async function flagPostFlight() {
		const sid = sessionId ?? $policyCanvasSessionId;
		if (!sid) return;

		const reason = flagReason.trim();
		if (!reason) return;

		flagSubmitting = true;
		try {
			const res = await fetch(`${apiBase}/planning/${sid}/confirm`, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ confirm: false, flag_reason: reason })
			});

			if (!res.ok) {
				error = `Failed to flag: ${res.status}`;
				return;
			}

			await fetchSession();
			showFlagInput = false;
			flagReason = '';
			error = null;
		} catch (e) {
			error = `Connection error: ${e.message}`;
		} finally {
			flagSubmitting = false;
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
			<!-- Post-flight state -->
			<div class="space-y-4">
				{#if session.post_flight}
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
							Formalization complete
						</p>
					</div>

					<p class="text-sm text-gray-600 dark:text-gray-400">
						{session.post_flight.summary}
					</p>

					{#if session.post_flight.status === 'pending_confirm'}
						{#if showFlagInput}
							<div class="space-y-2">
								<label
									class="text-xs font-medium text-gray-500 dark:text-gray-400"
								>
									Reason for flagging:
								</label>
								<textarea
									bind:value={flagReason}
									rows="2"
									class="w-full text-sm px-3 py-1.5 rounded border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800 text-gray-900 dark:text-gray-100 placeholder-gray-400 dark:placeholder-gray-500 focus:outline-none focus:ring-1 focus:ring-red-500"
									placeholder="Describe the issue..."
								/>
								<div class="flex gap-2">
									<button
										class="flex-1 py-2 text-sm font-medium rounded bg-red-600 hover:bg-red-700 text-white disabled:opacity-50 transition-colors"
										disabled={!flagReason.trim() || flagSubmitting}
										on:click={flagPostFlight}
									>
										{flagSubmitting ? 'Submitting...' : 'Submit Flag'}
									</button>
									<button
										class="px-3 py-2 text-sm rounded border border-gray-200 dark:border-gray-700 text-gray-600 dark:text-gray-400 hover:bg-gray-50 dark:hover:bg-gray-800 transition-colors"
										on:click={() => {
											showFlagInput = false;
											flagReason = '';
										}}
									>
										Cancel
									</button>
								</div>
							</div>
						{:else}
							<div class="flex gap-2 pt-2">
								<button
									class="flex-1 py-2 text-sm font-medium rounded-lg bg-green-600 hover:bg-green-700 dark:bg-green-500 dark:hover:bg-green-600 text-white disabled:opacity-50 transition-colors"
									disabled={confirmSubmitting}
									on:click={confirmPostFlight}
								>
									{confirmSubmitting ? 'Confirming...' : 'Confirm & Commit'}
								</button>
								<button
									class="px-3 py-2 text-sm font-medium rounded-lg border border-amber-300 dark:border-amber-700 text-amber-700 dark:text-amber-400 hover:bg-amber-50 dark:hover:bg-amber-900/20 transition-colors"
									on:click={() => {
										showFlagInput = true;
									}}
								>
									Flag
								</button>
							</div>
						{/if}
					{:else if session.post_flight.status === 'confirmed'}
						<div
							class="flex items-center gap-2 px-3 py-2 rounded-lg bg-green-50 dark:bg-green-900/20 text-green-700 dark:text-green-400"
						>
							<svg
								xmlns="http://www.w3.org/2000/svg"
								viewBox="0 0 20 20"
								fill="currentColor"
								class="w-4 h-4"
							>
								<path
									fill-rule="evenodd"
									d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z"
									clip-rule="evenodd"
								/>
							</svg>
							<span class="text-sm font-medium">Confirmed and committed</span>
						</div>
					{:else if session.post_flight.status === 'flagged'}
						<div
							class="px-3 py-2 rounded-lg bg-amber-50 dark:bg-amber-900/20 text-amber-700 dark:text-amber-400"
						>
							<p class="text-sm font-medium">Flagged for review</p>
							{#if session.post_flight.flag_reason}
								<p class="text-xs mt-1">{session.post_flight.flag_reason}</p>
							{/if}
						</div>
					{/if}
				{:else}
					<p class="text-sm text-gray-500 dark:text-gray-400">
						Waiting for post-flight summary from agent...
					</p>
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
