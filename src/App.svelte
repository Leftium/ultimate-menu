<script type='text/coffeescript'>
    import Main from './Main.svelte'
    import Configure from './Configure.svelte'

    component = Main

    if location.pathname isnt '/'
        component = Configure

</script>


<svelte:component this={component} />

<style>
    :global(body) {
        overflow-x: hidden;
    }
</style>
