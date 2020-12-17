<script type='text/coffeescript'>
export href=''
export aid=''
export bid=''
</script>

<a target='_blank' href='{href.replace(/:BID/g, bid).replace(/:AID/g, aid)}'>
    <slot></slot>
</a>
