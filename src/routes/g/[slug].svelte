<script lang="ts">
import { browser } from '$app/env';
import { page } from '$app/stores'
import { store } from '$lib/stores';

import { CustomLocalStorage } from '$lib/customLocalStorage';
import { Struct } from '$lib/struct.class';
import { Rights } from '$lib/rights.class';
import { JsonParser } from '$lib/jsonParser';
import { FactoryTimeline } from '$lib/factoryTimeline';
import { get } from '$lib/timelineRepository';

import Draw from '$lib/Draw.svelte';
import Toast from '$lib/Toast.svelte';
import { NotFoundOnlineException } from '$lib/timelineException.class';

let toastComponent
$store.rights = new Rights($page.url.searchParams)


//TODO : disabling SSR or not on this page ?
// OR
//TODO : using SSR to display loading gif
// https://kit.svelte.dev/docs#hooks-handle
// OR
//TODO : using SSR to retrive information from fauna


//TODO : displaying link & qrcode ?
// https://medium.com/geekculture/few-ways-to-generate-qr-code-using-javascript-54b6b5220c4f


const slug = $page.params.slug
if(slug.endsWith(".png")){
    console.error("An image was misconfigurated, eg bad = 'foo.png', good = '/foo.png'")
    toastComponent.show("An image was misconfigurated, eg bad = 'foo.png', good = '/foo.png'", false, 0)
}
let currentTimeline: Struct.Timeline = CustomLocalStorage.getTimeline(slug)

//If the local copie of Timeline has bigger rights than current url query parameter 
//  We refresh the window.location with the higher rights
let queryString = null
if(!$store.rights.hasOwner() && currentTimeline?.ownerKey){
    queryString = "?o=" + currentTimeline.ownerKey
} else if(!$store.rights.hasWriter() && currentTimeline?.writeKey){
    queryString = "?w=" + currentTimeline.writeKey
} else if(!$store.rights.hasReader() && currentTimeline?.readKey){
    queryString = "?r=" + currentTimeline.readKey
}
if(queryString){
    window.location.href = $page.url.protocol + '//' + $page.url.host + "/g/" + currentTimeline.key + queryString
}

if($store.rights.isNone()){
    if(!currentTimeline && browser){
        currentTimeline = new Struct.Timeline(slug, "My new Project")
        currentTimeline = FactoryTimeline.initiate(currentTimeline)
    }
    $store.currentTimeline = currentTimeline
} else {
    let seachParams = new URLSearchParams([['key', slug], [$store.rights.getTimelineField(), $store.rights.getSlugParamKeyValue()]])
    get(seachParams).then((json)=>{    

        if(!json["message"]["data"]) {
            console.error('node data not found in json["message"] : %o', json["message"])    
            throw new Error('node data not found in json["message"]')
        }

        currentTimeline = JSON.parse(JSON.stringify(json["message"]["data"]), JsonParser.timelineReviver) 
          

        //We're using the tricks of cloning the content of the Svelte store to avoid multiple refresh of store
        let cloneStore = {...$store}
        //Update date of lastUpdated in the clone
        cloneStore.lastUpdatedLocally = null
        cloneStore.lastCommitedRemotely = json["message"]["ts"]
        cloneStore.currentTimeline = currentTimeline
        // Tricks : Set to true if we don't want to refresh lastUpdatedLocally property
        cloneStore._cancelRefreshLastUpdatedLocally = true
        store.set(cloneStore)

    }).catch((err) => {
        console.error("Error where calling get() in [slug].svelte : %o", err)
        if(toastComponent){
            if (err instanceof NotFoundOnlineException) {
                toastComponent.show("Oups, your timeline doesn't seems to exist anymore on the remote endpoint.<br/> Please remove it from your local browser in the homepage", false, 10)
                //Refresh page after 10s
                setTimeout(function(){
                    window.location.href = $page.url.protocol + '//' + $page.url.host + "/g/" + slug
                }, 10000);
                
            } else {
                toastComponent.show("Oups, we couldn't reach the remote endpoint so we'll use your local data instead.<br/> Please check your browser console for more informations. Click me to dismiss the notif", false, 0)

            }
        }
        $store.currentTimeline = currentTimeline 
    }).finally(()=>{
        
    })
}
            
</script>

<svelte:head>
    <title>[T-C] {$store.currentTimeline?$store.currentTimeline.title:'Please wait a second'}</title>
</svelte:head>

{#if $store.currentTimeline}
<Draw/>
{/if}


<Toast bind:this={toastComponent}/>