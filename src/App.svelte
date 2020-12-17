<script type='text/coffeescript'>
    import { onMount, tick } from 'svelte'

    import { Temporal } from 'proposal-temporal'

    import * as birch from 'birch-outline'

    import Select from 'svelte-select'

    config = '''
Businesses:
	🔒 Leftium @business(335226930254766)
		MC6135 @payment

		Ad Account 01 @account(626486004739350)
			 MC6135 @payment

	🔒 Amazing Health Science @business(3992912430736092)
		MC4750 @payment
		VISA1111 @payment

		AHS01 @account(2797337333703347)
			VISA1111 @payment
		AHS02 @account(1144959642572168)
			VISA1111 @payment
		AHS03 @account(302053170773369)

	John Murphy 1 @business(413115236387462)
		JM01 @account(365373097889567)

	🔒 John Murphy 2 @business(148595963673002)

Links:
	Businesses @url(https://business.facebook.com/select)
	Ad Manager @url(https://business.facebook.com/adsmanager/manage/campaigns?act=:AID&business_id=:BID)
	FB Blueprint @url(https://www.facebookblueprint.com/)
	Workplace @url(https://ultimatefreedom667.workplace.com/)
	Inner Circle @url(https://innercircle.ultimatefreedom.com/)
'''

    outline = new birchoutline.Outline.createTaskPaperOutline(config)
    window.outline = outline

    businesses     = []
    accounts       = []
    paymentMethods = []

    selectItems = []
    groupBy = (item) -> item.group
    selectedValue = undefined

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


    console.log businessItems

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

    console.log businesses
    console.log accounts
    console.log paymentMethods
    console.log selectItems

    currentBusinessAccounts = accounts.filter (account) ->
        account.business is currentBusiness
    currentBusiness = businesses[0]
    currentAccount = accounts[0] or EMPTY_ACCOUNT

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

        currentBusiness = businesses[0]
        currentBusinessAccounts = accounts.filter (account) ->
            account.business is currentBusiness

        currentAccount = accounts[0] or EMPTY_ACCOUNT

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
        currentBusiness = businesses.find (b) -> b.id is newBusinessId
        currentAccount = currentBusiness.accounts[0] or EMPTY_ACCOUNT

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
        currentBusiness = businesses.find (b) -> b.id is currentAccount.business.id


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

        await tick()  # important
        selectedValue = undefined


</script>

<template lang=pug>
main
    h2 LA time: {laNowString}
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
    hr


    div
        +each('businesses as business')
            div.hidden-container
                span(bid='{business.id}' class='business' class:selected='{business.id == currentBusiness.id}' on:click='{handleClickBusiness}') {business.name}
                span(bid='{business.id}' class='hidden-element hidden-menu-item' class:selected='{business.id == currentBusiness.id}' on:click='{handleClickBusinessId}') {business.id} (click to copy id)
            +each('business.accounts as account')
                div.hidden-container
                    span(aid='{account.id}'
                         class='account'
                         class:selected='{account.id == currentAccount.id}'
                         on:click='{handleClickAccount}') {account.name}
                    span(aid='{account.id}'
                         class='hidden-element hidden-menu-item'
                         class:selected='{account.id == currentAccount.id}'
                         on:click='{handleClickAccountId}') {account.id} (click to copy id)

    div.themed-select
        Select(bind:selectedValue
               class='select'
               placeholder='Search for account by name'
               on:select='{handleSelect}'
               items='{selectItems}'
               groupBy='{groupBy}')

    hr
    div: a(target='_blank' href='https://ultimatefreedom667.workplace.com/') Workplace
    div: a(target='_blank' href='https://business.facebook.com/adsmanager/manage/campaigns?act={currentAccount.id}&business_id={currentBusiness.id}') Ads Manager
    div: a(target='_blank' href='https://business.facebook.com/settings/info?business_id={currentBusiness.id}') Business Info
    div: a(target='_blank' href='https://business.facebook.com/settings/support_inbox?business_id={currentBusiness.id}&global_scope_id={currentBusiness.id}') Business Verification Inbox
    div: a(target='_blank' href='https://business.facebook.com/accountquality/{currentBusiness.id}/{currentAccount.id}/') Account Quality
    div: a(target='_blank' href='https://business.facebook.com/ads/manager/account_settings/account_billing/?act={currentAccount.id}&pid=p1&business_id={currentBusiness.id}&page=account_settings&tab=account_billing_settings') Payment Settings
    div: a(target='_blank' href='https://business.facebook.com/ads/manager/billing/transactions/?act={currentAccount.id}&pid=p1&business_id={currentBusiness.id}&global_scope_id={currentBusiness.id}&page=billing&tab=transactions') Invoices
    div: a(target='_blank' href='https://business.facebook.com/adsmanager/manage/accounts?act={currentAccount.id}&business_id={currentBusiness.id}') Account Overview
    div: a(target='_blank' href='https://status.fb.com/ads') FB Ads Status
    div: a(target='_blank' href='https://business.facebook.com/select') Select Business
    div: a(target='_blank' href='https://business.facebook.com/create') Create Business
    div: a(target='_blank' href='http://www.worldtimebuddy.com/?lid=1835848,5128581,5037649,5391959&h=1835848') World Time Buddy (EST CST PST)


</template>

<style>
    main {
        padding: 1.2em;
        max-width: 240px;
        margin: 0 auto;
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


    :global(.selected) {
        font-weight: bold;
    }

    @media (min-width: 640px) {
        main {
            max-width: none;
        }
    }
</style>
