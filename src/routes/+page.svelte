<script lang="ts">
    import CSSRuntimeProvider from "@master/css.svelte"
    import config from "../../master.css"
</script>

<CSSRuntimeProvider config={config}>
    <main></main>
</CSSRuntimeProvider>

<style>
    @import '@master/normal.css';
</style>
