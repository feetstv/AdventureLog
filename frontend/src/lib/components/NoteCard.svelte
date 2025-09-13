<script lang="ts">
	import { t } from 'svelte-i18n';
	import { addToast } from '$lib/toasts';
	import type { Collection, Note, User } from '$lib/types';
	import { createEventDispatcher } from 'svelte';
	const dispatch = createEventDispatcher();

	import Launch from '~icons/mdi/launch';
	import TrashCan from '~icons/mdi/trash-can';
	import DeleteWarning from './DeleteWarning.svelte';
	import { isEntityOutsideCollectionDateRange } from '$lib/dateUtils';

	export let note: Note;
	export let user: User | null = null;
	export let collection: Collection | null = null;

	let isWarningModalOpen: boolean = false;
	let outsideCollectionRange: boolean = false;

	$: {
		if (collection) {
			outsideCollectionRange = isEntityOutsideCollectionDateRange(note, collection);
		}
	}

	function editNote() {
		dispatch('edit', note);
	}

	async function deleteNote() {
		const res = await fetch(`/api/notes/${note.id}`, {
			method: 'DELETE'
		});
		if (res.ok) {
			addToast('success', $t('notes.note_deleted'));
			isWarningModalOpen = false;
			dispatch('delete', note.id);
		} else {
			addToast($t('notes.note_delete_error'), 'error');
		}
	}
</script>

{#if isWarningModalOpen}
	<DeleteWarning
		title={$t('adventures.delete_note')}
		button_text="Delete"
		description={$t('adventures.note_delete_confirm')}
		is_warning={false}
		on:close={() => (isWarningModalOpen = false)}
		on:confirm={deleteNote}
	/>
{/if}

<div class="card w-full max-w-md bg-base-300 text-base-content shadow-2xl transition-all duration-300 border border-base-300 hover:border-primary/20 group p-4">
	<!-- Row 1: image | title/badges | detail (none) -->
	<div class="grid grid-cols-[auto,1fr,auto] items-center gap-3">
		<!-- Small circular image/icon -->
		<div class="flex items-center">
			<div class="avatar">
				<div class="w-12 h-12 rounded-full ring-1 ring-base-200 overflow-hidden">
					<div class="w-12 h-12 flex items-center justify-center text-xl bg-base-200">🗒️</div>
				</div>
			</div>
		</div>

		<!-- Title + badges -->
		<div class="col-span-1 col-start-2 text-left">
			<h2 class="font-semibold truncate">{note.name}</h2>
			<div class="mt-1 flex flex-wrap gap-1">
				<div class="badge badge-sm badge-primary">{$t('adventures.note')}</div>
				{#if outsideCollectionRange}
					<div class="badge badge-sm badge-error">{$t('adventures.out_of_range')}</div>
				{/if}
			</div>
		</div>

		<!-- Detail column intentionally left blank for NoteCard -->
		<div class="col-span-1"></div>
	</div>

	<!-- Row 2: actions -->
	<div class="mt-3 pt-3 border-t border-base-300 flex items-center justify-between gap-2">
		<button class="btn btn-neutral btn-sm flex items-center gap-1" on:click={editNote}>
			<Launch class="w-5 h-5" />
			<span class="hidden sm:inline">{$t('notes.open')}</span>
		</button>
		<div class="flex-1"></div>
		{#if note.user == user?.uuid || (collection && user && collection.shared_with?.includes(user.uuid))}
			<button id="delete_adventure" data-umami-event="Delete Adventure" class="btn btn-secondary btn-sm flex items-center gap-1" on:click={() => (isWarningModalOpen = true)}>
				<TrashCan class="w-5 h-5" />
				<span class="hidden sm:inline">{$t('adventures.delete')}</span>
			</button>
		{/if}
	</div>
</div>
