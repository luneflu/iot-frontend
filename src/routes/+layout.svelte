<script lang="ts">
	import favicon from '#lib/assets/favicon.svg';
	import '../app.css';
	import { Toaster } from '$lib/components/ui/sonner/index.js';
	import type { LayoutProps } from './$types';

	let { children }: LayoutProps = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

<Toaster richColors position="top-right" />
{@render children()}
