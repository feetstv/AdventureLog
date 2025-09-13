<script lang="ts">
	import { createEventDispatcher } from 'svelte';
	import TrashCanOutline from '~icons/mdi/trash-can-outline';
	import FileDocumentEdit from '~icons/mdi/file-document-edit';
	import type { Collection, Transportation, User } from '$lib/types';
	import { addToast } from '$lib/toasts';
	import { t } from 'svelte-i18n';
	import DeleteWarning from './DeleteWarning.svelte';
	// import ArrowDownThick from '~icons/mdi/arrow-down-thick';
	import { TRANSPORTATION_TYPES_ICONS } from '$lib';
	import {
		formatAllDayDate,
		formatDateInTimezone,
		isEntityOutsideCollectionDateRange
	} from '$lib/dateUtils';
	import { isAllDay } from '$lib';


	function getTransportationIcon(type: string) {
		if (type in TRANSPORTATION_TYPES_ICONS) {
			return TRANSPORTATION_TYPES_ICONS[type as keyof typeof TRANSPORTATION_TYPES_ICONS];
		} else {
			return '🚗';
		}
	}
	const dispatch = createEventDispatcher();

	export let transportation: Transportation;
	export let user: User | null = null;
	export let collection: Collection | null = null;

	const toMiles = (km: any) => (Number(km) * 0.621371).toFixed(1);

	let isWarningModalOpen: boolean = false;

	function editTransportation() {
		dispatch('edit', transportation);
	}

	let outsideCollectionRange: boolean = false;

	$: {
		if (collection) {
			outsideCollectionRange = isEntityOutsideCollectionDateRange(transportation, collection);
		}
	}

	async function deleteTransportation() {
		let res = await fetch(`/api/transportations/${transportation.id}`, {
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
			dispatch('delete', transportation.id);
		}
	}
</script>

{#if isWarningModalOpen}
	<DeleteWarning
		title={$t('adventures.delete_transportation')}
		button_text="Delete"
		description={$t('adventures.transportation_delete_confirm')}
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
					{#if transportation.images && transportation.images.length > 0}
						<img src={transportation.images[0]} alt={transportation.name} class="object-cover w-12 h-12" />
					{:else}
						<div class="w-12 h-12 flex items-center justify-center text-xl bg-base-200">{getTransportationIcon(transportation.type)}</div>
					{/if}
				</div>
			</div>
		</div>

		<!-- Title + badges -->
		<div class="col-span-1 col-start-2 text-left">
			<h2 class="font-semibold truncate">{transportation.name}</h2>
			<div class="mt-1 flex flex-wrap gap-1">
				<div class="badge badge-sm badge-secondary">{$t(`transportation.modes.${transportation.type}`)} {getTransportationIcon(transportation.type)}</div>
				{#if transportation.type === 'plane' && transportation.flight_number}
					<div class="badge badge-sm badge-neutral">{transportation.flight_number}</div>
				{/if}
				{#if outsideCollectionRange}
					<div class="badge badge-sm badge-error">{$t('adventures.out_of_range')}</div>
				{/if}
			</div>
		</div>

		<!-- Single detail: start date/time -->
		<div class="col-span-1 text-right text-sm">
			{#if transportation.date}
				<span>
					{#if isAllDay(transportation.date) && (!transportation.end_date || isAllDay(transportation.end_date))}
						{formatAllDayDate(transportation.date)}
					{:else}
						{formatDateInTimezone(transportation.date, transportation.start_timezone)}
						{#if transportation.start_timezone}
							<span class="ml-1 text-xs opacity-60">({transportation.start_timezone})</span>
						{/if}
					{/if}
				</span>
			{/if}
		</div>
	</div>

	<!-- Row 2: actions -->
	{#if transportation.user === user?.uuid || (collection && user && collection.shared_with?.includes(user.uuid))}
		<div class="mt-3 pt-3 border-t border-base-300 flex items-center justify-between gap-2">
			<button class="btn btn-neutral btn-sm flex items-center gap-1" on:click={editTransportation} title={$t('transportation.edit')}>
				<FileDocumentEdit class="w-5 h-5" />
				<span class="hidden sm:inline">{$t('transportation.edit')}</span>
			</button>
			<div class="flex-1"></div>
			<button class="btn btn-secondary btn-sm flex items-center gap-1" on:click={() => (isWarningModalOpen = true)} title={$t('adventures.delete')}>
				<TrashCanOutline class="w-5 h-5" />
				<span class="hidden sm:inline">{$t('adventures.delete')}</span>
			</button>
		</div>
	{/if}
</div>
