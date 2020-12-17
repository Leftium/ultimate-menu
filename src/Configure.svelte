<script type='text/coffeescript'>
    import { writable, readable, derived } from 'svelte-persistent-store/dist/local'

    config = writable 'config', ''

</script>


<template lang=pug>
    div: a(href='/') Home
    textarea(bind:value='{$config}' spellcheck='false')
</template>

<style>
    textarea {
        width: 100%;
        height: 800px;
        white-space: nowrap;
        overflow: auto;
        font-family: Consolas,Monaco,Lucida Console,Liberation Mono,DejaVu Sans Mono,Bitstream Vera Sans Mono,Courier New, monospace;
    }

</style>
