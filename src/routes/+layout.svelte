<script lang="ts">
    import '../app.css';
	import CSSRuntimeProvider from '@master/css.svelte';
	import config from '../../master.css';
    let { children } = $props();
</script>

<CSSRuntimeProvider {config}>
<main class="w:100dvw h:100dvh">
    {@render children?.()}
</main>
</CSSRuntimeProvider>