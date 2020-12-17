<script type='text/coffeescript'>
    import { onMount, tick } from 'svelte'

    import { Temporal } from 'proposal-temporal'

    import * as birch from 'birch-outline'

    import Select from 'svelte-select'
    import FacebookBusinessLink from './components/FacebookBusinessLink.svelte'

    import { writable, readable, derived } from 'svelte-persistent-store/dist/local'

    import defaultConfig from './cs/default-config.coffee'

    config = writable 'config', defaultConfig

    if not $config.length
        config.set defaultConfig

    outline = new birchoutline.Outline.createTaskPaperOutline($config)
    window.outline = outline

    businesses     = []
    accounts       = []
    paymentMethods = []

    selectItems = []
    groupBy = (item) -> item.group
    selectedValue = undefined

    links = []

    EMPTY_ACCOUNT  =
        name: 'no accounts'
        id: 0

    upsertBusiness = (name, id) ->
        if not business = businesses.find (b) -> b.id is id
            business =
                name: name
                id: id
                accounts: []
            businesses.push business
        business

    upsertAccount = (name, id, business) ->
        if not account = accounts.find (a) -> a.id is id
            account =
                name: name
                id: id
                business: business
            accounts.push account
            business.accounts.push account
        account

    upsertPaymentMethod = (name, payerId) ->
        if not paymentMethod = paymentMethods.find (p) -> p.name is name
            paymentMethod =
                name: name
                usedBy: []
            paymentMethods.push paymentMethod
        paymentMethod.usedBy.push payerId
        paymentMethod

    businessItems = null

    for item in outline.items
        # console.log item
        # console.log item.bodyString
        # console.log item.getAttribute('data-type')
        # console.log item.attributeNames

        if item.getAttribute('data-type') is 'project' and item.bodyString.match /^Businesses:/
            businessItems = item.descendants

        if item.getAttribute('data-type') is 'project' and item.bodyString.match /^Links:/
            linkItems = item.descendants

    console.log businessItems
    console.log linkItems

    for item in businessItems
        console.log item.bodyString
        if businessId = item.getAttribute 'data-business'
            lastBusiness = upsertBusiness(item.bodyContentString, businessId)
            lastPayerId = lastBusiness.id

        if accountId = item.getAttribute 'data-account'
            lastAccount = upsertAccount(item.bodyContentString, accountId, lastBusiness)
            lastPayerId = lastAccount.id

        console.log item.hasAttribute('data-payment')
        if item.hasAttribute 'data-payment'
            lastPaymentMethod = upsertPaymentMethod(item.bodyContentString, lastPayerId)

    for account in accounts
        selectItems.push item =
            value: account.id
            label: account.name
            group: account.business.name

    for item in linkItems
        console.log item.bodyString
        if item.getAttribute('data-type') is 'project'
            links.push link =
                category: item.bodyContentString
        else
            links.push link =
                url: item.getAttribute 'data-url'
                text: item.bodyContentString


    currentBusinessIdPersist = writable 'currentBusinessId', 0
    currentAccountIdPersist = writable 'currentAccountId', 0

    console.log businesses
    console.log accounts
    console.log paymentMethods
    console.log selectItems

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

    groupNumbers = (s) ->
        s.replace(/..../g, '$& ').trim()


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
            # second: '2-digit'
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

    handleClickBusiness = (e) ->
        newBusinessId = @.getAttribute 'bid'
        currentBusinessIdPersist.set newBusinessId
        currentBusiness = businesses.find (b) -> b.id is newBusinessId
        currentAccountIdPersist.set(currentBusiness.accounts[0]?.id or 0)

    handleClickBusinessId = (e) ->
        businessId = @.getAttribute 'bid'
        listener = (e) ->
            e.clipboardData.setData 'text/html', businessId
            e.clipboardData.setData 'text/plain', businessId
            e.preventDefault()

        document.addEventListener 'copy', listener
        document.execCommand 'copy'
        document.removeEventListener 'copy', listener

    handleClickAccount = (e) ->
        newAccountId = @.getAttribute 'aid'
        currentAccount = accounts.find (a) -> a.id is newAccountId
        currentAccountIdPersist.set(newAccountId)

        currentBusiness = businesses.find (b) -> b.id is currentAccount.business.id
        currentBusinessIdPersist.set currentBusiness.id


    handleClickAccountId = (e) ->
        businessId = @.getAttribute 'aid'
        listener = (e) ->
            e.clipboardData.setData 'text/html', businessId
            e.clipboardData.setData 'text/plain', businessId
            e.preventDefault()

        document.addEventListener 'copy', listener
        document.execCommand 'copy'
        document.removeEventListener 'copy', listener

    handleSelect = (e) ->
        newAccountId = e.detail.value
        currentAccount = accounts.find (a) -> a.id is newAccountId
        currentBusiness = businesses.find (b) -> b.id is currentAccount.business.id

        currentAccountIdPersist.set newAccountId
        currentBusinessIdPersist.set currentBusiness.id

        await tick()  # important
        selectedValue = undefined


</script>

<template lang=pug>
main
    div: a(href='configure') Configure
    h2 LA time: {laNowString}

    div.label Linear Estimator
    div.inline-block(draggable='false')
        label { Math.round(percent) }%
        input(bind:this='{passedInput}'
              class:selected='{fixedInput === passedInput}'
              value='0'
              on:focus='{handleFocus}'
              on:input='{handleInput}'
              draggable='false')
    div.inline-block
        label { 100-Math.round(percent) }%
        input(bind:this='{remainInput}'
              class:selected='{fixedInput === remainInput}'
              value='0'
              on:focus='{handleFocus}'
              on:input='{handleInput}')
    div.inline-block
        label 100%
        input(bind:this='{totalInput}'
              class:selected='{fixedInput === totalInput}'
              on:focus='{handleFocus}'
              on:input='{handleInput}'
              value=24)
    div.label Business/Account Selector (for links below)

    div
        +each('businesses as business')
            div.hidden-container
                span(bid='{business.id}' class='business' class:selected='{business.id == $currentBusinessIdPersist}' on:click='{handleClickBusiness}') {business.name}
                span(bid='{business.id}' class='hidden-element hidden-menu-item' class:selected='{business.id == $currentBusinessIdPersist}' on:click='{handleClickBusinessId}') {groupNumbers(business.id)} (click to copy id)
            +each('business.accounts as account')
                div.hidden-container
                    span(aid='{account.id}'
                         class='account'
                         class:selected='{account.id == $currentAccountIdPersist}'
                         on:click='{handleClickAccount}') {account.name}
                    span(aid='{account.id}'
                         class='hidden-element hidden-menu-item'
                         class:selected='{account.id == $currentAccountIdPersist}'
                         on:click='{handleClickAccountId}') {groupNumbers(account.id)} (click to copy id)

    div.themed-select
        Select(bind:selectedValue
               class='select'
               placeholder='Search for account by name'
               on:select='{handleSelect}'
               items='{selectItems}'
               groupBy='{groupBy}')

    div.label Links
    +each('links as link')
        +if('link.category')
            div.category {link.category}
            +else('')
                div: FacebookBusinessLink(bid='{$currentBusinessIdPersist}'
                                          aid='{$currentAccountIdPersist}'
                                          href='{link.url}') {link.text}



</template>

<style>
    main {
        padding: 1.2em;
        max-width: 240px;
        margin: 0 auto;
    }

    .label {
        font-weight: bold;
        font-size: 110%;
        color: white;
        background-color: gray;
    }

    h1 {
        color: #ff3e00;
        text-transform: uppercase;
        font-size: 4em;
        font-weight: 100;
    }

    .hidden-container .hidden-element {
        display: none;
        cursor: default;
    }

    .hidden-container:hover .hidden-element {
        display:inline;
        cursor: default;
    }

    .hidden-menu-item {
        padding-left: 10px;
        font-size: 80%;
        color: blue;
        text-decoration: underline;
        font-weight: normal;
    }

    .inline-block {
        display: inline-block;
    }

    .themed-select {
        --listMaxHeight: 800px;
    }

    .business {
        cursor: default;
        font-size: 120%;
    }

    .account {
        cursor: default;
        padding-left: 20px;
    }

    .category {
        margin-top: 10px;
        font-size: 120%;
        font-weight: bold;
        color: darkcyan;
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
