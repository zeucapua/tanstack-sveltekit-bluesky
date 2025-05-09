<script lang="ts">
	import '../app.css';
	import { QueryClient, QueryClientProvider } from "@tanstack/svelte-query";
	import { SvelteQueryDevtools } from "@tanstack/svelte-query-devtools";

	let { children } = $props();

	// DEMO 0: Initial setup
	const queryClient = new QueryClient();
</script>

<main class="bg-orange-100 w-full min-w-screen min-h-screen font-serif p-2 md:p-8">
	<section class="p-4">
		<h1 class="text-3xl font-bold">TanStack in SvelteKit: Examples for Query and Table</h1>
		<h2 class="text-xl font-medium">
			Demo website for Svelte Summit Talk, presented by
			<a href="https://zeu.dev" class="underline">zeu.dev</a>
		</h2>

		<span class="flex gap-4">
			<a href="/" class="border px-3 py-2 cursor-pointer">Home</a>
			<a href="/search" class="border px-3 py-2 cursor-pointer">Search</a>
			<a href="/p/zeu.dev" class="border px-3 py-2 cursor-pointer">Profile (default <code>zeu.dev</code>)</a>
		</span>
	</section>

	<QueryClientProvider client={queryClient}>
		<SvelteQueryDevtools />
		<span class="flex flex-col gap-8 p-8">
			{@render children()}
		</span>
	</QueryClientProvider>

</main>
