<script type='text/coffeescript'>
    import { onMount } from 'svelte'
    import { Temporal } from 'proposal-temporal'

    laNowString = null
    percent     = null
    hoursPassed = null
    hoursRemain = null
    total       = 24

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
    updateTime()

    onMount ->
        interval = setInterval updateTime, 1000

        # Clean up when destroyed.
        () -> clearInterval interval

</script>

<template lang=pug>
main
    h2 LA time: {laNowString}
    span
        input(value='{ percent.toFixed(1) }')
    span
        input(value='{ hoursPassed.toFixed(1) }')
    span
        input(value='{ hoursRemain.toFixed(1) }')
    span
        input(value='{ total.toFixed(1) }')

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

    @media (min-width: 640px) {
        main {
            max-width: none;
        }
    }
</style>
