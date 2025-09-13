<script lang="ts">
	import { createEventDispatcher } from 'svelte';
	import { goto } from '$app/navigation';
	import type { Location, Collection, User } from '$lib/types';
	const dispatch = createEventDispatcher();

	import Launch from '~icons/mdi/launch';
	import FileDocumentEdit from '~icons/mdi/file-document-edit';
	import TrashCan from '~icons/mdi/trash-can-outline';
	import MapMarker from '~icons/mdi/map-marker';
	import { addToast } from '$lib/toasts';
	import Link from '~icons/mdi/link-variant';
	import LinkVariantRemove from '~icons/mdi/link-variant-remove';
	import Plus from '~icons/mdi/plus';
	import CollectionLink from './CollectionLink.svelte';
	import DotsHorizontal from '~icons/mdi/dots-horizontal';
	import DeleteWarning from './DeleteWarning.svelte';
	import { t } from 'svelte-i18n';
	import { isEntityOutsideCollectionDateRange } from '$lib/dateUtils';

	export let type: string | null = null;
	export let user: User | null;
	export let collection: Collection | null = null;
	export let readOnly: boolean = false;

	let isCollectionModalOpen: boolean = false;
	let isWarningModalOpen: boolean = false;

	export let adventure: Location;

	let outsideCollectionRange: boolean = false;

	$: {
		if (collection) {
			outsideCollectionRange = adventure.visits.every((visit) =>
				isEntityOutsideCollectionDateRange(visit, collection)
			);
		}
	}

	async function deleteAdventure() {
		let res = await fetch(`/api/locations/${adventure.id}`, {
			method: 'DELETE'
		});
		if (res.ok) {
			addToast('info', $t('adventures.location_delete_success'));
			dispatch('delete', adventure.id);
		} else {
			console.log('Error deleting adventure');
		}
	}

	async function linkCollection(event: CustomEvent<string>) {
		let collectionId = event.detail;
		const updatedCollections = adventure.collections ? [...adventure.collections] : [];
		if (!updatedCollections.some((c) => String(c) === String(collectionId))) {
			updatedCollections.push(collectionId);
		}
		let res = await fetch(`/api/locations/${adventure.id}`, {
			method: 'PATCH',
			headers: { 'Content-Type': 'application/json' },
			body: JSON.stringify({ collections: updatedCollections })
		});
		if (res.ok) {
			adventure.collections = updatedCollections;
			addToast('info', `${$t('adventures.collection_link_location_success')}`);
		} else {
			addToast('error', `${$t('adventures.collection_link_location_error')}`);
		}
	}

	async function removeFromCollection(event: CustomEvent<string>) {
		let collectionId = event.detail;
		if (!collectionId) {
			addToast('error', `${$t('adventures.collection_remove_location_error')}`);
			return;
		}
		if (adventure.collections) {
			const updatedCollections = adventure.collections.filter((c) => String(c) !== String(collectionId));
			let res = await fetch(`/api/locations/${adventure.id}`, {
				method: 'PATCH',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ collections: updatedCollections })
			});

			if (res.ok) {
				adventure.collections = updatedCollections;
				addToast('info', `${$t('adventures.collection_remove_location_success')}`);
			} else {
				addToast('error', `${$t('adventures.collection_remove_location_error')}`);
			}
		}
	}

	function editAdventure() {
		dispatch('edit', adventure);
	}

	function link() {
		dispatch('link', adventure);
	}
</script>

{#if isCollectionModalOpen}
	<CollectionLink
		on:link={(e) => linkCollection(e)}
		on:unlink={(e) => removeFromCollection(e)}
		on:close={() => (isCollectionModalOpen = false)}
		linkedCollectionList={adventure.collections}
	/>
{/if}

{#if isWarningModalOpen}
	<DeleteWarning
		title={$t('adventures.delete_location')}
		button_text="Delete"
		description={$t('adventures.location_delete_confirm')}
		is_warning={false}
		on:close={() => (isWarningModalOpen = false)}
		on:confirm={deleteAdventure}
	/>
{/if}

<div class="card w-full max-w-md bg-base-300 shadow-2xl transition-all duration-300 border border-base-300 hover:border-primary/20 group p-4">
	<!-- Row 1: image | title/badges | detail -->
	<div class="grid grid-cols-[auto,1fr,auto] items-center gap-3">
		<!-- Small circular image/icon -->
		<div class="flex items-center">
			<div class="avatar">
				<div class="w-12 h-12 rounded-full ring-1 ring-base-200 overflow-hidden">
					{#if adventure.images && adventure.images.length > 0}
						<img src={adventure.images[0]} alt={adventure.name} class="object-cover w-12 h-12" />
					{:else}
						<div class="w-12 h-12 flex items-center justify-center text-xl bg-base-200">
							{adventure.category?.icon || '📍'}
						</div>
					{/if}
				</div>
			</div>
		</div>

		<!-- Title + badges -->
		<div class="col-span-1 col-start-2 text-left">
			<a href="/locations/{adventure.id}" class="font-semibold hover:text-primary line-clamp-2 block">
				{adventure.name}
			</a>
			<!-- Badges under title -->
			<div class="mt-1 flex flex-wrap gap-1">
				<div class="badge badge-sm {adventure.is_visited ? 'badge-success' : 'badge-warning'}">
					{adventure.is_visited ? $t('adventures.visited') : $t('adventures.planned')}
				</div>
				{#if adventure.category}
					<div class="badge badge-sm badge-primary">
						{adventure.category.display_name} {adventure.category.icon}
					</div>
				{/if}
				{#if outsideCollectionRange}
					<div class="badge badge-sm badge-error">{$t('adventures.out_of_range')}</div>
				{/if}
			</div>
		</div>

		<!-- Single detail: location -->
		<div class="col-span-1 text-right">
			{#if adventure.location}
				<div class="text-sm flex items-center justify-end gap-1 text-base-content/80">
					<MapMarker class="w-4 h-4 text-primary" />
					<span class="truncate max-w-[10rem]">{adventure.location}</span>
				</div>
			{/if}
		</div>
	</div>

	<!-- Row 2: actions -->
	{#if !readOnly}
		<div class="mt-3 pt-3 border-t border-base-300 flex items-center justify-between gap-2">
			{#if type != 'link'}
				<button class="btn btn-base-300 btn-sm" on:click={() => goto(`/locations/${adventure.id}`)}>
					<Launch class="w-4 h-4" />
					<span class="hidden sm:inline">{$t('adventures.open_details')}</span>
				</button>
				<div class="flex-1"></div>
				{#if (adventure.user && adventure.user.uuid == user?.uuid) || (collection && user && collection.shared_with?.includes(user.uuid)) || (collection && user && collection.user == user.uuid)}
					<div class="dropdown dropdown-end">
						<div tabindex="0" role="button" class="btn btn-square btn-sm btn-base-300">
							<DotsHorizontal class="w-5 h-5" />
						</div>
						<!-- svelte-ignore a11y-no-noninteractive-tabindex -->
						<ul tabindex="0" class="dropdown-content menu bg-base-100 rounded-box z-[1] w-56 p-2 shadow-xl border border-base-300">
							<li>
								<button on:click={editAdventure} class="flex items-center gap-2">
									<FileDocumentEdit class="w-4 h-4" />
									<span class="hidden sm:inline">{$t('adventures.edit_location')}</span>
								</button>
							</li>
							{#if user?.uuid == adventure.user?.uuid}
								<li>
									<button on:click={() => (isCollectionModalOpen = true)} class="flex items-center gap-2">
										<Plus class="w-4 h-4" />
										<span class="hidden sm:inline">{$t('collection.manage_collections')}</span>
									</button>
								</li>
							{:else if collection && user && collection.user == user.uuid}
								<li>
									<button on:click={() =>
										removeFromCollection(new CustomEvent('unlink', { detail: collection.id }))
									} class="flex items-center gap-2">
										<LinkVariantRemove class="w-4 h-4" />
										<span class="hidden sm:inline">{$t('adventures.remove_from_collection')}</span>
									</button>
								</li>
							{/if}
							{#if user.uuid == adventure.user?.uuid}
								<div class="divider my-1"></div>
								<li>
									<button id="delete_adventure" data-umami-event="Delete Adventure" class="text-error flex items-center gap-2" on:click={() => (isWarningModalOpen = true)}>
										<TrashCan class="w-4 h-4" />
										<span class="hidden sm:inline">{$t('adventures.delete')}</span>
									</button>
								</li>
							{/if}
						</ul>
					</div>
				{/if}
			{:else}
				<button class="btn btn-primary btn-sm" on:click={link}>
					<Link class="w-4 h-4" />
					<span class="hidden sm:inline">Link Adventure</span>
				</button>
			{/if}
		</div>
	{/if}
</div>

<style>
	.line-clamp-2 {
		display: -webkit-box;
		-webkit-line-clamp: 2;
		line-clamp: 2;
		-webkit-box-orient: vertical;
		overflow: hidden;
	}
</style>
