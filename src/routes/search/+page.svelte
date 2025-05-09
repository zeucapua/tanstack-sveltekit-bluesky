<script lang="ts">
	import { page } from "$app/state";
	import { atpAgent } from "$lib/atproto";
	import type { ProfileView } from "@atproto/api/dist/client/types/app/bsky/actor/defs";
	import { createInfiniteQuery } from "@tanstack/svelte-query";
	import { Debounced } from "runed";

	let searchTerm = $state(page.url.searchParams.get("q") || "");
	const debouncedSearchTerm = new Debounced(() => searchTerm, 500);

	// DEMO: Query Key as Dependency + Infinite Query
	const searchQuery = $derived(createInfiniteQuery({
		queryKey: ["searchActors", debouncedSearchTerm.current],
		staleTime: 30000,
		initialPageParam: "",
		queryFn: async () => {
			const { data } = await atpAgent.searchActors({ q: debouncedSearchTerm.current });
			return data as { actors: ProfileView[], cursor: string };
		},
		getNextPageParam: (lastPage) => lastPage.cursor,
		select: (data) => data.pages.map((page) => page.actors).flat()
	}));
</script>

<h1 class="text-3xl font-bold">Search Profiles</h1>
<a href="https://github.com/zeucapua/tanstack-sveltekit-bluesky/blob/main/src/routes/search/%2Bpage.svelte" class="flex gap-4 bg-black text-white px-3 py-2 w-fit h-fit">
	GitHub
</a>

<input
	bind:value={searchTerm}
	type="text"
	placeholder="alice.bsky.social"
	class="border px-3 py-2 mb-4"
/>

{#if !searchTerm}
	<p>Search by a user's handle (eg. <code>zeu.dev</code>)</p>
{:else if $searchQuery.isFetching}
	<p>Loading...</p>
{:else if $searchQuery.isError}
	<p class="text-red">{$searchQuery.error.message}</p>
{:else if $searchQuery.isSuccess}
	{#each $searchQuery.data as profile: ProfileView}
		{@render searchProfileDisplay(profile.handle, profile.avatar, profile.displayName)}
	{/each}

	{#if $searchQuery.hasNextPage}
		<button
			onclick={() => $searchQuery.fetchNextPage()}
			disabled={$searchQuery.isFetchingNextPage}
			class="flex justify-center gap-2 px-3 py-2 border disabled:bg-neutral-200"
		>
			{#if $searchQuery.isFetchingNextPage}
				<p class="animate-spin">↻</p>
			{/if}
			Add Next Page
		</button>
	{/if}
{/if}

{#snippet searchProfileDisplay(handle: string, avatar: string | undefined, displayName: string | undefined)}
	<a href={`/p/${handle}`} class="border p-4 cursor-pointer">
		<img src={avatar} alt={`${handle} avatar`} class="size-16 float-left mr-4" />
		<p class="text-lg font-bold wrap-anywhere">{displayName || "N/A"}</p>
		<p>@{handle}</p>
	</a>
{/snippet}
