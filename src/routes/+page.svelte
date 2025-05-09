<script lang="ts">
	import { untrack } from "svelte";
	import { Debounced } from "runed";
	import { atpAgent } from "$lib/atproto";
	import ProfileCell from "$lib/ProfileCell.svelte";
	import type { ProfileView, ProfileViewDetailed } from "@atproto/api/dist/client/types/app/bsky/actor/defs";
	import type { FeedViewPost, PostView } from "@atproto/api/dist/client/types/app/bsky/feed/defs";

	// TanStack Imports
	import { createInfiniteQuery, createQuery } from "@tanstack/svelte-query";

	// Thank you Simon H aka @dummdidumm for `tanstack-table-8-svelte-5@^0.1`!
	// Used as a drop in replacement
	// The current official release of TanStack Table library has
	// `createSvelteTable` importing from `svelte/internal`
	// which will cause an error because it's a server-side only package.
	import {
		// necessities
		createColumnHelper,
		createSvelteTable,
		getCoreRowModel,

		// sorting
		getSortedRowModel,
		type SortingState,
		type OnChangeFn,

		// custom components
		flexRender,
		renderComponent,
		getFilteredRowModel,
	} from "@tanstack/svelte-table";

	// --- TANSTACK QUERY EXAMPLES ---

	// --- DEMO 1: Basic Query
	// Fetch Zeu's profile
	const zeuQuery = createQuery({
		// `queryKey` is the unique identifier for each query
		queryKey: ["zeuProfile"],

		// `queryFn` is a Promise based method
		// that either resolves or throws an error
		queryFn: async () => {
			const { data } = await atpAgent.getProfile({ actor: "zeu.dev" });
			return data;
		},
	});

	// DEMO 2: Query Key as Dependency
	// Fetch a profile based on given handle
	const profileQuery = (handle: string) => createQuery({
		// we can add a variable as part of the `queryKey` array
		// to differentiate between similar queries
		queryKey: ["profile", handle],
		queryFn: async () => {
			const { data } = await atpAgent.getProfile({ actor: handle });
			return data;
		},
		staleTime: 30000 // in milliseconds, how fresh a data fetch is (cached)
	});

	// DEMO 3: Make Query Reactive
	// Fetch profile based on handle input
	let handleInput = $state("bsky.app");
	const debouncedHandle = new Debounced(() => handleInput, 500);

	// wrap the Query in `$derived`
	// whenever the key changes, Query will automatically trigger a refetch
	let reactiveProfileQuery = $derived(profileQuery(debouncedHandle.current));

	// DEMO 4: Infinite Query
	// Fetch and paginate the Svelte Feed by Paolo Ricciuti
	const svelteFeedQuery = createInfiniteQuery({
		queryKey: ["svelteFeed"],

		initialPageParam: "",

		// `pageParam` = what is set by `initialPageParam` and `getNextPageParam`
		queryFn: async ({ pageParam }) => {
			const { data } = await atpAgent.app.bsky.feed.getFeed({
				feed: "at://did:plc:ezyrzvz3yoglekd4j2szmiys/app.bsky.feed.generator/svelte-feed",
				limit: 10,
				cursor: pageParam
			});

			return data as { feed: FeedViewPost[], cursor: string };
		},

		// `lastPage` = what is returned by `queryFn`
		getNextPageParam: (lastPage) => lastPage.cursor,

		// `select` transforms the data returned by `queryFn`
		select: (data) =>
			// since this is an infinite query, the data is in pages
			// in our case, we want to turn the pages into a flat array of posts
			data.pages.map((page) => page.feed).flat()
	});

	// --- TANSTACK TABLE EXAMPLE ---

	// DEMO 5: Basic Table elements
	// Fetch the searched profile's followers list
	const followingQuery = $derived(createInfiniteQuery({
		queryKey: ["following", debouncedHandle.current],
		staleTime: 30000,
		initialPageParam: "",
		queryFn: async ({ pageParam }) => {
			const { data } = await atpAgent.getFollowers({
				actor: debouncedHandle.current,
				cursor: pageParam
			});
			return data as { followers: ProfileView[], cursor: string };
		},
		getNextPageParam: (lastPage) => lastPage.cursor,
		select: (data) => data.pages.map((page) => page.followers).flat()
	}));

	const profileColumnHelper = createColumnHelper<ProfileView>();
	// `.display` = NO DATA, can't be used to sort, filter, etc.
	// `.accessor` = Based on data, can be used to sort, filter, etc.
	// `.group` = no data themselves, just groups other columns together.

	// define the columns and its logic for its given row's cell
	const defaultColumns = [
		// Handle
		profileColumnHelper.accessor("handle", {
			header: "Handle", // show in `thead tr`
			cell: (info) => info.getValue(), // what is returned for the row's cell
			filterFn: "includesString" // can either be custom or a built-in filter function
		}),

		// Avatar + DisplayName
		profileColumnHelper.accessor("displayName", {
			header: "Name",
			cell: ({ cell, row }) =>
				// use fn to render a custom component
				renderComponent(
					// a `.svelte` component
					ProfileCell,
					// props for `ProfileCell`
					// access other data of the same row as cell
					{ avatar: row.original.avatar!, displayName: cell.getValue()! }
				),
			enableSorting: false
		}),

		// Created At (Date)
		profileColumnHelper.accessor("createdAt", {
			header: "Created At",
			cell: (info) => info.getValue(),

			// can set a custom sorting function or use built-in options like `datetime`
			sortingFn: "datetime"
		}),
	];

	// --- DEMO 6: Sorting
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

	// --- DEMO 7: Filter
	let tableSearchTerm = $state("");

	// for `createSvelteTable`
	let options = $derived({
		columns: defaultColumns,
		data: $followingQuery.data || [],
		getCoreRowModel: getCoreRowModel(),

		// sorting
		getSortedRowModel: getSortedRowModel(),
		onSortingChange: setSorting,
		state: { sorting },

		// filter
		getFilteredRowModel: getFilteredRowModel()
	});

	$effect(() => {
		// will cause the $effect to run whenever term changes
		tableSearchTerm;

		// `untrack` will let this function run inside an `$effect`
		// but not when its dependencies, esp `$table`, changes
		untrack(() => $table.setGlobalFilter(tableSearchTerm));
	});

	// `$derived` makes it reactive based on the options,
	// in this case with the Query data being passed through
	const table = $derived(createSvelteTable(options));
</script>

<details open class="flex flex-col gap-4 p-4 border">
	<summary class="flex flex-col md:flex-row md:justify-between md:items-center gap-4 mb-8">
		<div>
			<h1 class="text-2xl font-bold">Basic Query</h1>
			<h2 class="text-lg italic">Fetch Zeu's profile</h2>
		</div>

		<a href="https://github.com" class="flex gap-4 bg-black text-white px-3 py-2 w-fit h-fit">
			GitHub
		</a>
	</summary>

	<!-- DEMO 1: Basic Query rendering -->
	{#if $zeuQuery.isFetching}
		<p>Loading...</p>
	{:else if $zeuQuery.isSuccess}
		{@const profile = $zeuQuery.data}
		{@render detailedProfileDisplay(profile)}
	{/if}

</details>

<details open class="flex flex-col gap-4 p-4 border">
	<summary class="flex flex-col md:flex-row md:justify-between md:items-center gap-4 mb-8">
		<div>
			<h1 class="text-2xl font-bold">Query Key as Dependency</h1>
			<h2 class="text-lg italic">Fetch profile based on given handle</h2>
		</div>

		<a href="https://github.com" class="flex gap-4 bg-black text-white px-3 py-2 w-fit h-fit">
			GitHub
		</a>
	</summary>

	<!-- DEMO 3: Make Query Reactive -->
	<input bind:value={handleInput} placeholder="bsky.app" class="border px-3 py-2 mb-4"/>

	<!-- DEMO 2: Query Key as Dependency -->
	{#if $reactiveProfileQuery.isFetching}
		<p>Loading...</p>
	{:else if $reactiveProfileQuery.isError}
		<p class="text-red-500">{$reactiveProfileQuery.error.message}</p>
	{:else if $reactiveProfileQuery.isSuccess}
		{@const profile = $reactiveProfileQuery.data}
		<div>
			{@render detailedProfileDisplay(profile)}
		</div>
	{/if}

</details>

<details open class="flex flex-col gap-4 p-4 border">
	<summary class="flex flex-col md:flex-row md:justify-between md:items-center gap-4 mb-8">
		<div>
			<h1 class="text-2xl font-bold">Infinite Queries</h1>
			<h2 class="text-lg italic">Fetch the Svelte Feed by Paolo Ricciuti</h2>
		</div>

		<a href="https://github.com" class="flex gap-4 bg-black text-white px-3 py-2 w-fit h-fit">
			GitHub
		</a>
	</summary>

	<!-- DEMO 4: Infinite Query -->
	{#if $svelteFeedQuery.isFetching}
		<p>Loading...</p>
	{:else if $svelteFeedQuery.isSuccess}
		{#each $svelteFeedQuery.data as { post }: { post: PostView }}
			{@render postDisplay(post)}
		{/each}

		<!-- Manual fetch next page -->
		<button
			onclick={() => $svelteFeedQuery.fetchNextPage()}
			disabled={$svelteFeedQuery.isFetchingNextPage}
			class="flex justify-center gap-2 px-3 py-2 border disabled:bg-neutral-200"
		>
			{#if $svelteFeedQuery.isFetchingNextPage}
				<p class="animate-spin">↻</p>
			{/if}
			Add Next Page
		</button>
	{/if}

</details>

<details open class="flex flex-col gap-4 p-4 border">
	<summary class="flex flex-col md:flex-row md:justify-between md:items-center gap-4 mb-8">
		<div>
			<h1 class="text-2xl font-bold">Table</h1>
			<h2 class="text-lg italic">Sort, Search, and Select with Infinite Query</h2>
		</div>

		<a href="https://github.com" class="flex gap-4 bg-black text-white px-3 py-2 w-fit h-fit">
			GitHub
		</a>
	</summary>

	<!-- Set current page -->
	<menu class="flex flex-col md:flex-row gap-4 mb-4">
		<!-- Manual fetch next page -->
		<button
			onclick={() => $followingQuery.fetchNextPage()}
			disabled={$followingQuery.isFetchingNextPage}
			class="flex justify-center gap-2 px-3 py-2 border disabled:bg-neutral-200"
		>
			{#if $followingQuery.isFetchingNextPage}
				<p class="animate-spin">↺</p>
			{/if}
			Fetch More Data
		</button>

		<!-- Set search term -->
		<input
			type="text"
			placeholder="Search table..."
			bind:value={tableSearchTerm}
			class="px-3 py-2 border"
		/>
	</menu>


	<!-- Render UI using the `$table` derived state -->
	<table class="table-auto border border-collapse w-full">
		<thead>
			<tr>
				{#each $table.getHeaderGroups() as headerGroup}
					{#each headerGroup.headers as header}
						<th class="border px-4 py-2 w-fit">
							<!-- DEMO 6: Sorting -->
							<button
								disabled={!header.column.getCanSort()}
								onclick={header.column.getToggleSortingHandler()}
								class="flex gap-2 items-center"
							>
								{header.column.columnDef.header}
								{#if header.column.getCanSort()}
									{#if header.column.getIsSorted().toString() === "asc"}
										<img src="sortAsc.svg" alt="Sort Ascending" />
									{:else if header.column.getIsSorted().toString() === "desc"}
										<img src="sortDesc.svg" alt="Sort Descending" />
									{:else}
										<img src="sortNone.svg" alt="Sort By" />
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
</details>

<!-- SNIPPETS -->

{#snippet basicProfileDisplay(handle: string, avatar: string | undefined, displayName: string | undefined)}
	<img src={avatar} alt={`${handle} avatar`} class="size-16 float-left mr-4" />
	<p class="text-lg font-bold break-words wrap-anywhere">{displayName || "N/A"}</p>
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
		{@render basicProfileDisplay(author.handle, author.avatar, author.displayName)}
		<p class="break-all">{post.record.text}</p>
	</output>
{/snippet}
