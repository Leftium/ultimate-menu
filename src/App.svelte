<script type='text/coffeescript'>
    import { onMount } from 'svelte'
    import { Temporal } from 'proposal-temporal'

    import * as birch from 'birch-outline'

    DECIMALS = 2

    laNowString = ''
    percent     = 0
    passed      = 0
    remain      = 0
    total       = 100

    passedInput = null
    remainInput = null
    totalInput  = null
    fixedInput  = null


    updateValues = (lastInput) ->
        switch lastInput
            when passedInput
                totalInput.value = (100 * passedInput.value / percent)
                remainInput.value = (totalInput.value - passedInput.value)
            when remainInput
                totalInput.value = (100 * remainInput.value / (100 - percent))
                passedInput.value = (totalInput.value - remainInput.value)
            when totalInput
                passedInput.value = (totalInput.value * percent/100)
                remainInput.value = (remain = totalInput.value - passedInput.value)

        if lastInput isnt passedInput
            passedInput.value = parseFloat(passedInput.value, 10).toFixed(DECIMALS)
        if lastInput isnt remainInput
            remainInput.value = parseFloat(remainInput.value, 10).toFixed(DECIMALS)
        if lastInput isnt totalInput
            totalInput.value = parseFloat(totalInput.value, 10).toFixed(DECIMALS)


    updateTime = () ->
        laNow = Temporal.now.zonedDateTimeISO('America/Los_Angeles')

        laNowString = laNow.toInstant().toLocaleString 'en-US', options =
            timeZone: 'America/Los_Angeles'
            timeZoneName: 'short'
            weekday: 'short'
            month: 'short'
            day: 'numeric'
            hour: 'numeric'
            minute: '2-digit'
            second: '2-digit'
        laNowString = laNowString.replace ',', ''

        laMidnight = laNow.round options =
            smallestUnit: 'day'
            roundingMode: 'floor'
        duration = laMidnight.until laNow, { smallestUnit: 'minutes' }
        hoursPassed = duration.total { unit: 'hours' }
        hoursRemain = 24 - hoursPassed
        percent = hoursPassed/24*100

        updateValues(fixedInput)

    onMount ->
        fixedInput = totalInput
        updateValues fixedInput

        console.log fixedInput

        updateTime()
        interval = setInterval updateTime, 1000

        # Clean up when destroyed.
        () -> clearInterval interval

    handleFocus = (e) ->
        e.srcElement.select()
        fixedInput = e.srcElement


    handleInput = (e) ->
        v = parseFloat (@value or 0), 10
        updateValues e.srcElement



</script>

<template lang=pug>
main
    h2 LA time: {laNowString}
    span
        input(on:input='{handleInput}'
              value='{ percent.toFixed(DECIMALS) }')
    span
        input(bind:this='{passedInput}'
              class:selected='{fixedInput === passedInput}'
              value='0'
              on:focus='{handleFocus}'
              on:input='{handleInput}')
    span
        input(bind:this='{remainInput}'
              class:selected='{fixedInput === remainInput}'
              value='0'
              on:focus='{handleFocus}'
              on:input='{handleInput}')
    span
        input(bind:this='{totalInput}'
              class:selected='{fixedInput === totalInput}'
              on:focus='{handleFocus}'
              on:input='{handleInput}'
              value=24)

</template>

<style>
    main {
        text-align: center;
        padding: 1em;
        max-width: 240px;
        margin: 0 auto;
    }

    h1 {
        color: #ff3e00;
        text-transform: uppercase;
        font-size: 4em;
        font-weight: 100;
    }

    :global(.selected) {
        font-weight: bold;
    }

    @media (min-width: 640px) {
        main {
            max-width: none;
        }
    }
</style>
