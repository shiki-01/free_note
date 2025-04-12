<script lang="ts">
    import Icon from "@iconify/svelte";

    let { icon, class: className = '', onclick, children = null } = $props();
</script>

<div class="w:100% h:64px rel flex flex-col justify-content:center align-items:center">
    <button
        onclick={onclick}
        class="w:100% h:100% flex justify-content:center align-items:center pointer-events:auto {className}"
    >
        <Icon icon={icon} class="w:32px h:32px" />
    </button>
    {@render children?.()}
</div>
