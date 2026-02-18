<script lang="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';

	const START_HOUR = 7;
	const START_OFFSET = START_HOUR * 2;

	const HALF_HOUR_SLOTS = Array.from({ length: 48 }, (_, index) => {
		const actualIndex = (index + START_OFFSET) % 48;
		const hour = Math.floor(actualIndex / 2);
		const minute = actualIndex % 2 === 0 ? 0 : 30;
		return {
			index,
			label: `${String(hour).padStart(2, '0')}:${String(minute).padStart(2, '0')}`,
			isAfterMidnight: actualIndex < START_OFFSET
		};
	});

	let weekStart: Date;
	let isReady = false;

	type AgendaBlock = {
		id: string;
		dayIndex: number;
		startSlot: number;
		endSlot: number;
		title: string;
		color: string;
	};

	let weekBlocks: AgendaBlock[] = [];
	let dragDayIndex: number | null = null;
	let dragStartSlot: number | null = null;
	let dragCurrentSlot: number | null = null;
	let dragSelection: { dayIndex: number; startSlot: number; endSlot: number } | null = null;
	let dragBlock: AgendaBlock | null = null;
	let dragBlockPreview: { dayIndex: number; startSlot: number; endSlot: number } | null = null;
	let dragBlockDuration = 1;
	let dragBlockIsCopy = false;
	let suppressBlockClick = false;
	let editDialog: HTMLDialogElement | null = null;
	let editBlock: AgendaBlock | null = null;
	let editTitle = '';
	let editStartSlot = 0;
	let editEndSlot = 1;
	let dialogMode: 'create' | 'edit' = 'edit';
	let createSelection: { dayIndex: number; startSlot: number; endSlot: number } | null = null;

	const dayLabels = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'];

	function getWeekStart(date: Date) {
		const start = new Date(date);
		start.setHours(0, 0, 0, 0);
		const day = start.getDay();
		const diff = (day + 6) % 7;
		start.setDate(start.getDate() - diff);
		return start;
	}

	function addDays(date: Date, days: number) {
		const next = new Date(date);
		next.setDate(next.getDate() + days);
		return next;
	}

	const STORAGE_KEY = 'weekly-planner:weekly-blocks';
	const BLOCK_COLORS = ['#6366f1', '#10b981', '#f97316', '#ef4444', '#06b6d4', '#8b5cf6'];

	function loadWeekData() {
		if (!browser) return;
		const raw = localStorage.getItem(STORAGE_KEY);
		weekBlocks = raw ? (JSON.parse(raw) as AgendaBlock[]) : [];
	}

	function saveWeekData() {
		if (!browser) return;
		localStorage.setItem(STORAGE_KEY, JSON.stringify(weekBlocks));
	}

	function createId() {
		if (browser && 'randomUUID' in crypto) {
			return crypto.randomUUID();
		}
		return `${Date.now()}-${Math.random().toString(16).slice(2)}`;
	}

	function addBlock(
		dayIndex: number,
		startSlot: number,
		endSlot: number,
		title: string,
		colorOverride?: string
	) {
		const color = colorOverride ?? BLOCK_COLORS[Math.floor(Math.random() * BLOCK_COLORS.length)];
		const nextBlock: AgendaBlock = {
			id: createId(),
			dayIndex,
			startSlot,
			endSlot,
			title,
			color
		};
		weekBlocks = [...weekBlocks, nextBlock];
		saveWeekData();
	}

	function updateBlockTitle(blockId: string, title: string) {
		weekBlocks = weekBlocks.map((block) =>
			block.id === blockId ? { ...block, title } : block
		);
		saveWeekData();
	}

	function updateBlockDetails(blockId: string, updates: Partial<AgendaBlock>) {
		weekBlocks = weekBlocks.map((block) =>
			block.id === blockId ? { ...block, ...updates } : block
		);
		saveWeekData();
	}

	function removeBlock(blockId: string) {
		weekBlocks = weekBlocks.filter((block) => block.id !== blockId);
		saveWeekData();
	}

	function clearWeek() {
		weekBlocks = [];
		saveWeekData();
	}

	function getTimeLabel(slotIndex: number) {
		const actualIndex = (slotIndex + START_OFFSET) % 48;
		const hour = Math.floor(actualIndex / 2);
		const minute = actualIndex % 2 === 0 ? 0 : 30;
		return `${String(hour).padStart(2, '0')}:${String(minute).padStart(2, '0')}`;
	}

	function handleSlotPointerDown(dayIndex: number, slotIndex: number) {
		dragDayIndex = dayIndex;
		dragStartSlot = slotIndex;
		dragCurrentSlot = slotIndex;
		updateDragSelection();
	}

	function handleSlotPointerEnter(dayIndex: number, slotIndex: number) {
		if (dragDayIndex === null || dragStartSlot === null || dragDayIndex !== dayIndex) return;
		dragCurrentSlot = slotIndex;
		updateDragSelection();
	}

	function updateDragSelection() {
		if (dragDayIndex === null || dragStartSlot === null || dragCurrentSlot === null) {
			dragSelection = null;
			return;
		}
		const startSlot = Math.min(dragStartSlot, dragCurrentSlot);
		const endSlot = Math.max(dragStartSlot, dragCurrentSlot) + 1;
		dragSelection = { dayIndex: dragDayIndex, startSlot, endSlot };
	}

	function finalizeDragSelection() {
		if (dragBlock) return;
		if (!dragSelection) {
			resetDrag();
			return;
		}
		const { dayIndex, startSlot, endSlot } = dragSelection;
		dialogMode = 'create';
		createSelection = { dayIndex, startSlot, endSlot };
		editBlock = null;
		editTitle = '';
		editStartSlot = startSlot;
		editEndSlot = endSlot;
		editDialog?.showModal();
		resetDrag();
	}

	function openEditDialog(block: AgendaBlock) {
		dialogMode = 'edit';
		createSelection = null;
		editBlock = block;
		editTitle = block.title;
		editStartSlot = block.startSlot;
		editEndSlot = block.endSlot;
		editDialog?.showModal();
	}

	function closeEditDialog() {
		editDialog?.close();
		editBlock = null;
		createSelection = null;
	}

	function saveEditDialog() {
		const normalizedEnd = Math.max(editEndSlot, editStartSlot + 1);
		const title = editTitle.trim();
		if (!title) return;
		if (dialogMode === 'create' && createSelection) {
			addBlock(createSelection.dayIndex, editStartSlot, normalizedEnd, title);
		} else if (editBlock) {
			updateBlockDetails(editBlock.id, {
				title,
				startSlot: editStartSlot,
				endSlot: normalizedEnd
			});
		}
		closeEditDialog();
	}

	function deleteEditBlock() {
		if (!editBlock) return;
		removeBlock(editBlock.id);
		closeEditDialog();
	}


	function handleDialogClick(event: MouseEvent) {
		if (event.target === editDialog) {
			closeEditDialog();
		}
	}

	function handleDialogClose() {
		editBlock = null;
		createSelection = null;
		dialogMode = 'edit';
	}

	function resetDrag() {
		dragDayIndex = null;
		dragStartSlot = null;
		dragCurrentSlot = null;
		dragSelection = null;
	}

	function handleBlockPointerDown(block: AgendaBlock, event: PointerEvent) {
		event.preventDefault();
		suppressBlockClick = false;
		dragBlock = block;
		dragBlockIsCopy = event.ctrlKey || event.metaKey;
		dragBlockDuration = Math.max(1, block.endSlot - block.startSlot);
		dragBlockPreview = {
			dayIndex: block.dayIndex,
			startSlot: block.startSlot,
			endSlot: block.endSlot
		};
	}

	function handleWindowPointerMove(event: PointerEvent) {
		if (!dragBlock) return;
		const target = document.elementFromPoint(event.clientX, event.clientY) as HTMLElement | null;
		const slotElement = target?.closest('.slot-background') as HTMLElement | null;
		if (!slotElement) return;
		const dayIndex = Number(slotElement.dataset.dayIndex);
		const slotIndex = Number(slotElement.dataset.slotIndex);
		if (Number.isNaN(dayIndex) || Number.isNaN(slotIndex)) return;
		const maxStart = 48 - dragBlockDuration;
		const startSlot = Math.min(Math.max(slotIndex, 0), maxStart);
		const endSlot = startSlot + dragBlockDuration;
		dragBlockPreview = { dayIndex, startSlot, endSlot };
		suppressBlockClick = true;
	}

	function finalizeBlockDrag() {
		if (!dragBlock) return;
		if (dragBlockPreview && suppressBlockClick) {
			if (dragBlockIsCopy) {
				addBlock(
					dragBlockPreview.dayIndex,
					dragBlockPreview.startSlot,
					dragBlockPreview.endSlot,
					dragBlock.title,
					dragBlock.color
				);
			} else {
				updateBlockDetails(dragBlock.id, {
					dayIndex: dragBlockPreview.dayIndex,
					startSlot: dragBlockPreview.startSlot,
					endSlot: dragBlockPreview.endSlot
				});
			}
		}
		resetBlockDrag();
	}

	function resetBlockDrag() {
		dragBlock = null;
		dragBlockPreview = null;
		dragBlockDuration = 1;
		dragBlockIsCopy = false;
	}

	function handlePointerUp() {
		if (dragBlock) {
			finalizeBlockDrag();
			return;
		}
		finalizeDragSelection();
	}

	function handlePointerLeave() {
		if (dragBlock) {
			finalizeBlockDrag();
			return;
		}
		resetDrag();
	}

	function handleBlockClick(block: AgendaBlock) {
		if (suppressBlockClick) {
			suppressBlockClick = false;
			return;
		}
		openEditDialog(block);
	}

	onMount(() => {
		weekStart = getWeekStart(new Date());
		loadWeekData();
		isReady = true;
	});

	$: weekDays = weekStart
		? Array.from({ length: 7 }, (_, index) => addDays(weekStart, index))
		: [];

	$: endSlotOptions = Array.from({ length: 48 }, (_, index) => index + 1).filter(
		(slot) => slot > editStartSlot
	);

	$: if (editEndSlot <= editStartSlot) {
		editEndSlot = editStartSlot + 1;
	}
</script>

<svelte:window
	on:pointerup={handlePointerUp}
	on:pointercancel={handlePointerUp}
	on:pointermove={handleWindowPointerMove}
	on:mouseleave={handlePointerLeave}
/>

<div class="calendar" aria-label="Weekly calendar">
	<div class="corner"></div>
	{#each dayLabels as label}
		<div class="day-header">{label}</div>
	{/each}

	{#each HALF_HOUR_SLOTS as slot}
		<div
			class="time-label"
			class:time-label--empty={slot.index % 2 === 1}
			class:time-label--after-midnight={slot.isAfterMidnight}
		>
			{slot.index % 2 === 0 ? slot.label : ''}
		</div>
		{#each weekDays as _, dayIndex}
			<div
				class="slot-background"
				class:slot-background--after-midnight={slot.isAfterMidnight}
				data-day-index={dayIndex}
				data-slot-index={slot.index}
				on:mousedown|preventDefault={() => handleSlotPointerDown(dayIndex, slot.index)}
				on:mouseenter={() => handleSlotPointerEnter(dayIndex, slot.index)}
			></div>
		{/each}
	{/each}

	<div class="calendar-overlay">
		{#if dragSelection}
			<div
				class="selection-block"
				style={`grid-column: ${dragSelection.dayIndex + 2}; grid-row: ${dragSelection.startSlot + 2} / ${dragSelection.endSlot + 2};`}
			></div>
		{/if}

		{#if dragBlockPreview && dragBlock}
			<div
				class="agenda-block agenda-block--preview"
				style={`grid-column: ${dragBlockPreview.dayIndex + 2}; grid-row: ${dragBlockPreview.startSlot + 2} / ${dragBlockPreview.endSlot + 2}; background: ${dragBlock.color};`}
			>
				<div class="block-title">{dragBlock.title}</div>
				<div class="block-time">
					{getTimeLabel(dragBlockPreview.startSlot)} - {getTimeLabel(dragBlockPreview.endSlot)}
				</div>
			</div>
		{/if}

		{#each weekBlocks as block}
			<div
				class="agenda-block"
				class:agenda-block--dragging={dragBlock?.id === block.id}
				style={`grid-column: ${block.dayIndex + 2}; grid-row: ${block.startSlot + 2} / ${block.endSlot + 2}; background: ${block.color};`}
				on:pointerdown|stopPropagation={(event) => handleBlockPointerDown(block, event)}
				on:click={() => handleBlockClick(block)}
			>
				<div class="block-title">{block.title}</div>
				<div class="block-time">
					{getTimeLabel(block.startSlot)} - {getTimeLabel(block.endSlot)}
				</div>
			</div>
		{/each}
	</div>
</div>

<dialog
	class="edit-dialog"
	bind:this={editDialog}
	on:click={handleDialogClick}
	on:close={handleDialogClose}
>
	<form class="dialog-card" on:submit|preventDefault={saveEditDialog}>
		<header class="dialog-header">
			<h3>{dialogMode === 'create' ? 'New block' : 'Edit block'}</h3>
			<button type="button" class="icon-button" on:click={closeEditDialog} aria-label="Close">
				×
			</button>
		</header>
		<div class="dialog-body">
			<label class="field">
				<span>Title</span>
				<input type="text" bind:value={editTitle} placeholder="Block name" required />
			</label>

			<div class="time-grid">
				<label class="field">
					<span>Start</span>
					<select bind:value={editStartSlot}>
						{#each HALF_HOUR_SLOTS as slot}
							<option value={slot.index}>{slot.label}</option>
						{/each}
					</select>
				</label>
				<label class="field">
					<span>End</span>
					<select bind:value={editEndSlot}>
						{#each endSlotOptions as slotIndex}
							<option value={slotIndex}>{getTimeLabel(slotIndex)}</option>
						{/each}
					</select>
				</label>
			</div>
		</div>
		<footer class="dialog-actions">
			<div class="dialog-actions__left">
				{#if dialogMode === 'edit' && editBlock}
					<button type="button" class="danger" on:click={deleteEditBlock}>Delete</button>
				{/if}
			</div>
			<div class="dialog-actions__right">
				<button type="button" class="ghost" on:click={closeEditDialog}>Cancel</button>
				<button type="submit">
					{dialogMode === 'create' ? 'Add block' : 'Save changes'}
				</button>
			</div>
		</footer>
	</form>
</dialog>

<style>
	:global(html, body) {
		margin: 0;
		padding: 0;
		height: 100%;
		width: 100%;
		font-family: 'Inter', system-ui, -apple-system, sans-serif;
		background: #f4f6fb;
		color: #0f172a;
	}

	button {
		border: none;
		border-radius: 999px;
		padding: 0.55rem 1.25rem;
		font-weight: 600;
		cursor: pointer;
		transition: transform 0.2s ease, box-shadow 0.2s ease;
	}

	button:hover {
		transform: translateY(-1px);
		box-shadow: 0 6px 14px rgba(15, 23, 42, 0.12);
	}

	button.ghost {
		background: #ffffff;
		color: #334155;
		border: 1px solid #e2e8f0;
	}

	.calendar {
		display: grid;
		--header-row: 40px;
		grid-template-columns: 68px repeat(7, minmax(110px, 1fr));
		grid-template-rows: var(--header-row) repeat(48, minmax(0, 1fr));
		border: 1px solid #e2e8f0;
		border-radius: 0;
		overflow: hidden;
		width: 100vw;
		height: 100vh;
		position: relative;
	}

	.calendar-overlay {
		position: absolute;
		inset: 0;
		display: grid;
		grid-template-columns: 68px repeat(7, minmax(110px, 1fr));
		grid-template-rows: var(--header-row) repeat(48, minmax(0, 1fr));
		pointer-events: none;
	}

	.corner {
		background: #f8fafc;
		border-bottom: 1px solid #e2e8f0;
		height: var(--header-row);
	}

	.day-header {
		padding: 0.45rem 0.5rem;
		text-align: center;
		font-weight: 700;
		background: #f8fafc;
		border-bottom: 1px solid #e2e8f0;
		font-size: 0.85rem;
		height: var(--header-row);
	}

	.time-label {
		padding: 0.2rem 0.35rem;
		text-align: right;
		font-size: 0.68rem;
		color: #64748b;
		border-right: 1px solid #e2e8f0;
		background: #ffffff;
	}

	.time-label--empty {
		color: transparent;
	}

	.time-label--after-midnight {
		background: #f1f5f9;
		color: #94a3b8;
	}

	.slot-background {
		border-bottom: 1px solid #f1f5f9;
		border-right: 1px solid #f1f5f9;
		background: #ffffff;
		cursor: crosshair;
		position: relative;
	}

	.slot-background--after-midnight {
		background: #f8fafc;
	}

	.selection-block {
		z-index: 2;
		border-radius: 10px;
		background: rgba(99, 102, 241, 0.2);
		border: 1px dashed #6366f1;
		margin: 1px 4px;
		pointer-events: none;
	}

	.agenda-block {
		z-index: 3;
		border-radius: 10px;
		color: #ffffff;
		padding: 0.25rem 0.4rem;
		margin: 1px 4px;
		display: flex;
		flex-direction: column;
		justify-content: space-between;
		gap: 0.15rem;
		box-shadow: 0 6px 12px rgba(15, 23, 42, 0.16);
		cursor: pointer;
		overflow: hidden;
		pointer-events: auto;
	}

	.agenda-block--preview {
		opacity: 0.75;
		border: 1px dashed rgba(255, 255, 255, 0.7);
		box-shadow: 0 10px 16px rgba(15, 23, 42, 0.18);
		pointer-events: none;
	}

	.agenda-block--dragging {
		opacity: 0.35;
		filter: saturate(0.85);
	}

	.block-title {
		font-size: 0.74rem;
		font-weight: 600;
		line-height: 1.2;
	}

	.block-time {
		font-size: 0.62rem;
		opacity: 0.85;
	}

	.edit-dialog {
		border: none;
		padding: 0;
		background: transparent;
		max-width: 480px;
		width: min(92vw, 480px);
	}

	.edit-dialog::backdrop {
		background: rgba(15, 23, 42, 0.45);
		backdrop-filter: blur(2px);
	}

	.dialog-card {
		background: #ffffff;
		border-radius: 20px;
		padding: 1.5rem;
		box-shadow: 0 24px 60px rgba(15, 23, 42, 0.2);
		display: flex;
		flex-direction: column;
		gap: 1.25rem;
	}

	.dialog-header {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 1rem;
	}

	.dialog-header h3 {
		margin: 0;
		font-size: 1.2rem;
	}

	.icon-button {
		background: #f1f5f9;
		border-radius: 999px;
		width: 32px;
		height: 32px;
		border: none;
		font-size: 1.1rem;
		cursor: pointer;
		line-height: 1;
	}

	.dialog-body {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.field {
		display: flex;
		flex-direction: column;
		gap: 0.35rem;
		font-size: 0.85rem;
		color: #475569;
	}

	.field input,
	.field select {
		border: 1px solid #e2e8f0;
		border-radius: 12px;
		padding: 0.6rem 0.75rem;
		font-size: 0.95rem;
		color: #0f172a;
		background: #ffffff;
	}

	.time-grid {
		display: grid;
		grid-template-columns: repeat(2, minmax(0, 1fr));
		gap: 1rem;
	}

	.dialog-actions {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 0.75rem;
	}

	.dialog-actions__left,
	.dialog-actions__right {
		display: flex;
		gap: 0.75rem;
	}

	button.danger {
		background: #ef4444;
		color: #ffffff;
	}

	button.danger:hover {
		box-shadow: 0 8px 18px rgba(239, 68, 68, 0.25);
	}

	@media (max-width: 1024px) {
		.calendar {
			grid-template-columns: 56px repeat(7, minmax(96px, 1fr));
		}
	}

	@media (max-width: 860px) {
		.calendar {
			grid-template-columns: 56px repeat(7, minmax(130px, 1fr));
			overflow-x: auto;
		}

		.calendar::-webkit-scrollbar {
			height: 8px;
		}

		.calendar::-webkit-scrollbar-thumb {
			background: #cbd5f5;
			border-radius: 999px;
		}
	}

	@media (max-width: 640px) {
		button {
			flex: 1 1 auto;
			text-align: center;
		}
	}
</style>
