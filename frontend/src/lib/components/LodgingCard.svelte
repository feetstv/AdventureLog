<script lang="ts">
	import { createEventDispatcher } from 'svelte';
	import TrashCanOutline from '~icons/mdi/trash-can-outline';
	import FileDocumentEdit from '~icons/mdi/file-document-edit';
	import type { Collection, Lodging, User } from '$lib/types';
	import { addToast } from '$lib/toasts';
	import { t } from 'svelte-i18n';
	import DeleteWarning from './DeleteWarning.svelte';
	import { LODGING_TYPES_ICONS } from '$lib';
	import { formatDateInTimezone, isEntityOutsideCollectionDateRange } from '$lib/dateUtils';
	import { formatAllDayDate } from '$lib/dateUtils';
	import { isAllDay } from '$lib';

	const dispatch = createEventDispatcher();

	function getLodgingIcon(type: string) {
		if (type in LODGING_TYPES_ICONS) {
			return LODGING_TYPES_ICONS[type as keyof typeof LODGING_TYPES_ICONS];
		} else {
			return '🏨';
		}
	}
	export let lodging: Lodging;
	export let user: User | null = null;
	export let collection: Collection | null = null;

	let isWarningModalOpen: boolean = false;

	function editTransportation() {
		dispatch('edit', lodging);
	}

	let outsideCollectionRange: boolean = false;

	$: {
		if (collection) {
			outsideCollectionRange = isEntityOutsideCollectionDateRange(lodging, collection);
		}
	}

	async function deleteTransportation() {
		let res = await fetch(`/api/lodging/${lodging.id}`, {
			method: 'DELETE',
			headers: {
				'Content-Type': 'application/json'
			}
		});
		if (!res.ok) {
			console.log($t('transportation.transportation_delete_error'));
		} else {
			addToast('info', $t('transportation.transportation_deleted'));
			isWarningModalOpen = false;
			dispatch('delete', lodging.id);
		}
	}
</script>

{#if isWarningModalOpen}
	<DeleteWarning
		title={$t('adventures.delete_lodging')}
		button_text="Delete"
		description={$t('adventures.lodging_delete_confirm')}
		is_warning={false}
		on:close={() => (isWarningModalOpen = false)}
		on:confirm={deleteTransportation}
	/>
{/if}

<div class="card w-full max-w-md bg-base-300 text-base-content shadow-2xl transition-all duration-300 border border-base-300 hover:border-primary/20 group p-4">
	<!-- Row 1: image | title/badges | detail -->
	<div class="grid grid-cols-[auto,1fr,auto] items-center gap-3">
		<!-- Small circular image/icon -->
		<div class="flex items-center">
			<div class="avatar">
				<div class="w-12 h-12 rounded-full ring-1 ring-base-200 overflow-hidden">
					{#if lodging.images && lodging.images.length > 0}
						<img src={lodging.images[0]} alt={lodging.name} class="object-cover w-12 h-12" />
					{:else}
						<div class="w-12 h-12 flex items-center justify-center text-xl bg-base-200">{getLodgingIcon(lodging.type)}</div>
					{/if}
				</div>
			</div>
		</div>

		<!-- Title + badges -->
		<div class="col-span-1 col-start-2 text-left">
			<h2 class="font-semibold truncate">{lodging.name}</h2>
			<div class="mt-1 flex flex-wrap gap-1">
				{#if lodging.type}
					<div class="badge badge-sm badge-secondary">{$t(`lodging.${lodging.type}`)} {getLodgingIcon(lodging.type)}</div>
				{/if}
				{#if outsideCollectionRange}
					<div class="badge badge-sm badge-error">{$t('adventures.out_of_range')}</div>
				{/if}
			</div>
		</div>

		<!-- Single detail: check-in date/time -->
		<div class="col-span-1 text-right text-sm">
			{#if lodging.check_in}
				<span>
					{#if isAllDay(lodging.check_in)}
						{formatAllDayDate(lodging.check_in)}
					{:else}
						{formatDateInTimezone(lodging.check_in, lodging.timezone)}
						{#if lodging.timezone}
							<span class="ml-1 text-xs opacity-60">({lodging.timezone})</span>
						{/if}
					{/if}
				</span>
			{/if}
		</div>
	</div>

	<!-- Row 2: actions -->
	{#if lodging.user == user?.uuid || (collection && user && collection.shared_with?.includes(user.uuid))}
		<div class="mt-3 pt-3 border-t border-base-300 flex items-center justify-between gap-2">
			<button class="btn btn-neutral btn-sm flex items-center gap-1" on:click={editTransportation} title={$t('transportation.edit')}>
				<FileDocumentEdit class="w-5 h-5" />
				<span class="hidden sm:inline">{$t('transportation.edit')}</span>
			</button>
			<div class="flex-1"></div>
			<button on:click={() => (isWarningModalOpen = true)} class="btn btn-secondary btn-sm flex items-center gap-1" title={$t('adventures.delete')}>
				<TrashCanOutline class="w-5 h-5" />
				<span class="hidden sm:inline">{$t('adventures.delete')}</span>
			</button>
		</div>
	{/if}
</div>
