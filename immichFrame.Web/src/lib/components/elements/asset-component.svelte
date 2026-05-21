<script lang="ts">
	import type { AssetResponseDto } from '$lib/immichFrameApi';
	import * as api from '$lib/index';
	import ErrorElement from './error-element.svelte';
	import Asset from './asset.svelte';
	import type AssetComponent from './asset.svelte';
	import LoadingElement from './LoadingElement.svelte';
	import { fade } from 'svelte/transition';
	import { configStore } from '$lib/stores/config.store';
	import { Confetti } from 'svelte-confetti';
	import { slideshowStore } from '$lib/stores/slideshow.store';
	import { untrack } from 'svelte';

	api.init();

	type AssetTuple = [string, AssetResponseDto, api.AlbumResponseDto[]];

	interface Props {
		assets: AssetTuple[];
		nextAssets?: AssetTuple[];
		interval?: number;
		error?: boolean;
		loaded?: boolean;
		split?: boolean;
		hasBday?: boolean;
		showLocation?: boolean;
		showPhotoDate?: boolean;
		showImageDesc?: boolean;
		showPeopleDesc?: boolean;
		showTagsDesc?: boolean;
		showAlbumName?: boolean;
		imageFill?: boolean;
		imageZoom?: boolean;
		imagePan?: boolean;
		showInfo: boolean;
		playAudio?: boolean;
		onVideoWaiting?: () => void;
		onVideoPlaying?: () => void;
	}

	let {
		assets,
		nextAssets = [],
		interval = 20,
		error = false,
		loaded = false,
		split = false,
		hasBday = false,
		showLocation = true,
		showPhotoDate = true,
		showImageDesc = true,
		showPeopleDesc = true,
		showTagsDesc = true,
		showAlbumName = true,
		imageFill = false,
		imageZoom = false,
		imagePan = false,
		showInfo = $bindable(false),
		playAudio = false,
		onVideoWaiting = () => {},
		onVideoPlaying = () => {}
	}: Props = $props();
	let instantTransition = slideshowStore.instantTransition;
	let transitionDuration = $derived(
		$instantTransition ? 0 : ($configStore.transitionDuration ?? 1) * 1000
	);
	let transitionDelay = $derived($instantTransition ? 0 : transitionDuration / 2 + 25);
	let useTwoSlot = $derived(!!$configStore.preloadNeighbors);

	// Legacy-path refs (used by the {#key} fallback)
	let primaryAssetComponent = $state<AssetComponent | undefined>(undefined);
	let secondaryAssetComponent = $state<AssetComponent | undefined>(undefined);

	// Two-slot path state
	let slotAAssets = $state<AssetTuple[]>([]);
	let slotBAssets = $state<AssetTuple[]>([]);
	let visibleSlot = $state<'a' | 'b'>('a');
	let slotARefs = $state<(AssetComponent | undefined)[]>([]);
	let slotBRefs = $state<(AssetComponent | undefined)[]>([]);

	$effect(() => {
		const currentUrl = assets[0]?.[0];
		const incomingNext = nextAssets;
		const incomingCurrent = assets;
		if (!currentUrl) return;

		const aUrl = untrack(() => slotAAssets[0]?.[0]);
		const bUrl = untrack(() => slotBAssets[0]?.[0]);

		if (aUrl === currentUrl) {
			visibleSlot = 'a';
			slotBAssets = incomingNext;
		} else if (bUrl === currentUrl) {
			visibleSlot = 'b';
			// Defer the hidden slot's content swap until after the opacity transition
			// has been kicked off, so the fading-out slot doesn't change src mid-fade.
			setTimeout(() => {
				slotAAssets = incomingNext;
			}, 0);
		} else {
			// Initial mount or backward jump — flash is expected here.
			slotAAssets = incomingCurrent;
			slotBAssets = incomingNext;
			visibleSlot = 'a';
		}
	});

	export const pause = async () => {
		if (useTwoSlot) {
			for (const ref of [...slotARefs, ...slotBRefs]) await ref?.pause?.();
		} else {
			await primaryAssetComponent?.pause?.();
			await secondaryAssetComponent?.pause?.();
		}
	};

	export const play = async () => {
		if (useTwoSlot) {
			const active = visibleSlot === 'a' ? slotARefs : slotBRefs;
			for (const ref of active) await ref?.play?.();
		} else {
			await primaryAssetComponent?.play?.();
			await secondaryAssetComponent?.play?.();
		}
	};
</script>

{#if hasBday}
	<div
		class="z-[1000] top-[-50px] fixed l-0 h-dvh-safe w-screen flex justify-center overflow-hidden pointer-events-none"
	>
		<Confetti
			x={[-5, 5]}
			y={[0, 0.1]}
			delay={[500, 2000]}
			infinite
			duration={5000}
			amount={200}
			fallDistance="100vh"
		/>
	</div>
{/if}

{#if error}
	<ErrorElement />
{:else if loaded && useTwoSlot}
	<div class="grid absolute h-dvh-safe w-screen">
		{#if slotAAssets.length > 0}
			<div
				class="absolute inset-0 transition-opacity"
				class:opacity-0={visibleSlot !== 'a'}
				class:pointer-events-none={visibleSlot !== 'a'}
				style="transition-duration: {transitionDuration / 2}ms;"
			>
				{#if split && slotAAssets.length === 2}
					<div class="grid grid-cols-2 h-dvh-safe w-screen">
						<div class="relative grid border-r-2 border-primary h-dvh-safe">
							<Asset
								asset={slotAAssets[0]}
								{interval}
								{showLocation}
								{showPhotoDate}
								{showImageDesc}
								{showPeopleDesc}
								{showTagsDesc}
								{showAlbumName}
								{imageFill}
								{imageZoom}
								{imagePan}
								{split}
								{playAudio}
								{onVideoWaiting}
								{onVideoPlaying}
								bind:this={slotARefs[0]}
								bind:showInfo
							/>
						</div>
						<div class="relative grid border-l-2 border-primary h-dvh-safe">
							<Asset
								asset={slotAAssets[1]}
								{interval}
								{showLocation}
								{showPhotoDate}
								{showImageDesc}
								{showPeopleDesc}
								{showTagsDesc}
								{showAlbumName}
								{imageFill}
								{imageZoom}
								{imagePan}
								{split}
								{playAudio}
								{onVideoWaiting}
								{onVideoPlaying}
								bind:this={slotARefs[1]}
								bind:showInfo
							/>
						</div>
					</div>
				{:else}
					<div class="relative grid h-dvh-safe w-screen">
						<Asset
							asset={slotAAssets[0]}
							{interval}
							{showLocation}
							{showPhotoDate}
							{showImageDesc}
							{showPeopleDesc}
							{showTagsDesc}
							{showAlbumName}
							{imageFill}
							{imageZoom}
							{imagePan}
							{split}
							{playAudio}
							{onVideoWaiting}
							{onVideoPlaying}
							bind:this={slotARefs[0]}
							bind:showInfo
						/>
					</div>
				{/if}
			</div>
		{/if}
		{#if slotBAssets.length > 0}
			<div
				class="absolute inset-0 transition-opacity"
				class:opacity-0={visibleSlot !== 'b'}
				class:pointer-events-none={visibleSlot !== 'b'}
				style="transition-duration: {transitionDuration / 2}ms;"
			>
				{#if split && slotBAssets.length === 2}
					<div class="grid grid-cols-2 h-dvh-safe w-screen">
						<div class="relative grid border-r-2 border-primary h-dvh-safe">
							<Asset
								asset={slotBAssets[0]}
								{interval}
								{showLocation}
								{showPhotoDate}
								{showImageDesc}
								{showPeopleDesc}
								{showTagsDesc}
								{showAlbumName}
								{imageFill}
								{imageZoom}
								{imagePan}
								{split}
								{playAudio}
								{onVideoWaiting}
								{onVideoPlaying}
								bind:this={slotBRefs[0]}
								bind:showInfo
							/>
						</div>
						<div class="relative grid border-l-2 border-primary h-dvh-safe">
							<Asset
								asset={slotBAssets[1]}
								{interval}
								{showLocation}
								{showPhotoDate}
								{showImageDesc}
								{showPeopleDesc}
								{showTagsDesc}
								{showAlbumName}
								{imageFill}
								{imageZoom}
								{imagePan}
								{split}
								{playAudio}
								{onVideoWaiting}
								{onVideoPlaying}
								bind:this={slotBRefs[1]}
								bind:showInfo
							/>
						</div>
					</div>
				{:else}
					<div class="relative grid h-dvh-safe w-screen">
						<Asset
							asset={slotBAssets[0]}
							{interval}
							{showLocation}
							{showPhotoDate}
							{showImageDesc}
							{showPeopleDesc}
							{showTagsDesc}
							{showAlbumName}
							{imageFill}
							{imageZoom}
							{imagePan}
							{split}
							{playAudio}
							{onVideoWaiting}
							{onVideoPlaying}
							bind:this={slotBRefs[0]}
							bind:showInfo
						/>
					</div>
				{/if}
			</div>
		{/if}
	</div>
{:else if loaded}
	{#key assets}
		<div
			class="grid absolute h-dvh-safe w-screen"
			out:fade={{ duration: transitionDuration / 2 }}
			in:fade={{ duration: transitionDuration / 2, delay: transitionDelay }}
		>
			{#if split}
				<div class="grid grid-cols-2">
					<div id="image_portrait_1" class="relative grid border-r-2 border-primary h-dvh-safe">
						<Asset
							asset={assets[0]}
							{interval}
							{showLocation}
							{showPhotoDate}
							{showImageDesc}
							{showPeopleDesc}
							{showTagsDesc}
							{showAlbumName}
							{imageFill}
							{imageZoom}
							{imagePan}
							{split}
							{playAudio}
							{onVideoWaiting}
							{onVideoPlaying}
							bind:this={primaryAssetComponent}
							bind:showInfo
						/>
					</div>
					<div id="image_portrait_2" class="relative grid border-l-2 border-primary h-dvh-safe">
						<Asset
							asset={assets[1]}
							{interval}
							{showLocation}
							{showPhotoDate}
							{showImageDesc}
							{showPeopleDesc}
							{showTagsDesc}
							{showAlbumName}
							{imageFill}
							{imageZoom}
							{imagePan}
							{split}
							{playAudio}
							{onVideoWaiting}
							{onVideoPlaying}
							bind:this={secondaryAssetComponent}
							bind:showInfo
						/>
					</div>
				</div>
			{:else}
				<div id="image_default" class="relative grid h-dvh-safe w-screen">
					<Asset
						asset={assets[0]}
						{interval}
						{showLocation}
						{showPhotoDate}
						{showImageDesc}
						{showPeopleDesc}
						{showTagsDesc}
						{showAlbumName}
						{imageFill}
						{imageZoom}
						{imagePan}
						{split}
						{playAudio}
						{onVideoWaiting}
						{onVideoPlaying}
						bind:this={primaryAssetComponent}
						bind:showInfo
					/>
				</div>
			{/if}
		</div>
	{/key}
{:else}
	<div class="grid absolute h-dvh-safe w-screen">
		<LoadingElement />
	</div>
{/if}
