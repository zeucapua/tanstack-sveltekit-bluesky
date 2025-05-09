<script lang="ts">
	let { avatar, displayName }: { avatar: string, displayName: string } = $props();
</script>

<span class="flex items-center gap-2 w-full">
	<img src={avatar} alt={`${displayName} avatar`} class="size-8" />
	<p>{displayName}</p>
</span>
