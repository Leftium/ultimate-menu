<script type='text/coffeescript'>
    import { writable, readable, derived } from 'svelte-persistent-store/dist/local'

    config = writable 'config', ''

    handleKeyDown = (e) ->
        # https://stackoverflow.com/questions/6637341/use-tab-to-indent-in-textarea
        if e.key is 'Tab'
            e.preventDefault()
            start = @selectionStart
            end = @selectionEnd

            # Set textarea value to: text before caret + tab + text after caret.
            @value = @value.substring(0, start) + '    ' + @value.substring(end)

            # Put caret at right position again.
            @selectionStart = @selectionEnd = start + 4

</script>


<template lang=pug>
    div: a(href='/') Home
    textarea(bind:value='{$config}' on:keydown='{handleKeyDown}' spellcheck='false')
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
