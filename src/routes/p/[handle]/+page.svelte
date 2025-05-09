<script lang="ts">
	import { page } from "$app/state";
	import { atpAgent } from "$lib/atproto";
	import type { ProfileViewDetailed } from "@atproto/api/dist/client/types/app/bsky/actor/defs";
	import type { FeedViewPost, PostView } from "@atproto/api/dist/client/types/app/bsky/feed/defs";
	import { createInfiniteQuery, createQuery } from "@tanstack/svelte-query";
	import { createSvelteTable, flexRender } from "@tanstack/svelte-table";
	import { createColumnHelper, getCoreRowModel, getFilteredRowModel, getSortedRowModel, type OnChangeFn, type SortingState } from "@tanstack/table-core";
	import { untrack } from "svelte";

	const { handle } = page.params;

	// DEMO: Query Key as Dependency
	const profileQuery = createQuery({
		queryKey: ["profile", handle],
		queryFn: async () => {
			const { data } = await atpAgent.getProfile({ actor: handle });
			return data;
		},
		staleTime: 30000
	});

	// DEMO: Infinite Query
	const authorFeedQuery = createInfiniteQuery({
		queryKey: ["authorFeed", handle],
		staleTime: 30000,
		initialPageParam: "",
		queryFn: async ({ pageParam }) => {
			const { data } = await atpAgent.app.bsky.feed.getAuthorFeed({
				actor: handle,
				limit: 10,
				cursor: pageParam
			});

			return data as { feed: FeedViewPost[], cursor: string };
		},
		getNextPageParam: (lastPage) => lastPage.cursor,
		select: (data) => data.pages.map((page) => page.feed).flat()
			.filter((post: FeedViewPost) => !post.reason && !post.reply)
	});

	// DEMO: Feed as Table
	const postColumnHelper = createColumnHelper<FeedViewPost>();

	const defaultColumns = [
		postColumnHelper.accessor("post.likeCount", {
			header: "Likes",
			cell: (info) => info.getValue(),
			sortingFn: "alphanumeric"
		}),

		postColumnHelper.accessor("post.record.text", {
			header: "Text",
			cell: (info) => info.getValue(),
			filterFn: "includesString",
			enableSorting: false
		})
	];

	let sorting: SortingState = $state([]);

	const setSorting: OnChangeFn<SortingState> = updater => {
		// update SortingState
    if (updater instanceof Function) {
      sorting = updater(sorting)
    } else {
      sorting = updater
    }

		// update in `options` object
		options.state.sorting = sorting;
  }

	let tableSearchTerm = $state("");

	// for `createSvelteTable`
	let options = $derived({
		columns: defaultColumns,
		data: $authorFeedQuery.data || [],
		getCoreRowModel: getCoreRowModel(),

		// sorting
		getSortedRowModel: getSortedRowModel(),
		onSortingChange: setSorting,
		state: { sorting },

		// filter
		getFilteredRowModel: getFilteredRowModel()
	});

	$effect(() => {
		tableSearchTerm;
		untrack(() => $table.setGlobalFilter(tableSearchTerm));
	});

	const table = $derived(createSvelteTable(options));
</script>

<h1 class="text-3xl font-bold">{handle} Profile Page</h1>
<a href="https://github.com/zeucapua/tanstack-sveltekit-bluesky/blob/main/src/routes/p/%5Bhandle%5D/%2Bpage.svelte" class="flex gap-4 bg-black text-white px-3 py-2 w-fit h-fit">
	GitHub
</a>

{#if $profileQuery.isFetching}
	<p>Loading...</p>
{:else if $profileQuery.isError}
	<p class="text-red-500">{$profileQuery.error.message}</p>
{:else if $profileQuery.isSuccess}
	{@const profile = $profileQuery.data}
	{@render detailedProfileDisplay(profile)}

	<input
		type="text"
		placeholder="Search table..."
		bind:value={tableSearchTerm}
		class="px-3 py-2 border"
	/>

	<!-- Render UI using the `$table` derived state -->
	<table class="table-auto border border-collapse w-full">
		<thead>
			<tr>
				{#each $table.getHeaderGroups() as headerGroup}
					{#each headerGroup.headers as header}
						<th class="border px-4 py-2 w-fit">
							<button
								disabled={!header.column.getCanSort()}
								onclick={header.column.getToggleSortingHandler()}
								class="flex gap-2 items-center"
							>
								{header.column.columnDef.header}
								{#if header.column.getCanSort()}
									{#if header.column.getIsSorted().toString() === "asc"}
										<img src="/sortAsc.svg" alt="Sort Ascending" />
									{:else if header.column.getIsSorted().toString() === "desc"}
										<img src="/sortDesc.svg" alt="Sort Descending" />
									{:else}
										<img src="/sortNone.svg" alt="Sort By" />
									{/if}
								{/if}
							</button>
						</th>
					{/each}
				{/each}
		</thead>
		<tbody>
			{#each $table.getRowModel().rows as row}
				<tr>
					{#each row.getVisibleCells() as cell}
						<!-- How to render a custom component for a cell -->
						{@const Component = flexRender(cell.column.columnDef.cell, cell.getContext())}
						<td class="border w-fit">
							<Component />
						</td>
					{/each}
				</tr>
			{/each}
		</tbody>
	</table>

	<button
		onclick={() => $authorFeedQuery.fetchNextPage()}
		disabled={$authorFeedQuery.isFetchingNextPage}
		class="px-4 py-2 border rounded-sm"
	>
		{#if $authorFeedQuery.isFetchingNextPage}
			Loading...
		{:else}
			Load more...
		{/if}
	</button>
{/if}

{#snippet basicProfileDisplay(handle: string, avatar: string | undefined, displayName: string | undefined)}
	<img src={avatar} alt={`${handle} avatar`} class="size-16 float-left mr-4" />
	<p class="text-lg font-bold wrap-anywhere">{displayName || "N/A"}</p>
	<p>@{handle}</p>
{/snippet}

{#snippet detailedProfileDisplay(profile: ProfileViewDetailed)}
	<output>
		{@render basicProfileDisplay(
			profile.handle,
			profile.avatar,
			profile.displayName
		)}
		<p><b>Description</b>: {profile.description}</p>
	</output>
{/snippet}

{#snippet postDisplay(post: PostView)}
	{@const author = post.author}
	<output class="border p-2">
		<p class="break-all">{post.record.text}</p>
	</output>
{/snippet}
