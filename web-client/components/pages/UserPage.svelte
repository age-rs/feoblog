<!--
    Shows posts by a single user.
-->

<div class="item">
    <h1>Posts by:</h1>
    <div class="userInfo">
        <UserIDView {appState} {userID}/>
    </div>
    <Button href={`#/u/${userID}/profile`}>View Profile</Button>
</div>

{#each items as entry, index (entry.signature)}
    <ItemView 
        userID={entry.userID.toString()}
        signature={entry.signature.toString()}
        item={entry.item}
        {appState}
    />
{:else}
    <div class="item">No posts found for user <UserIDView {appState} {userID}/></div>
{/each}

<VisibilityCheck on:itemVisible={displayMoreItems} bind:visible={endIsVisible}/>

<script lang="ts">
import { tick } from "svelte/internal";

import type { Writable } from "svelte/store";

import type { Item, ItemList, ItemListEntry } from "../../protos/feoblog";
import type { AppState } from "../../ts/app";
import { UserID, Signature } from "../../ts/client";
import { ConsoleLogger, prefetch } from "../../ts/common";

import ItemView from "../ItemView.svelte"
import Button from "../Button.svelte"
import VisibilityCheck from "../VisibilityCheck.svelte";
import UserIDView from "../UserIDView.svelte"

export let appState: Writable<AppState>

let items: DisplayItem[] = []
let lazyItems: AsyncIterator<DisplayItem> = getDisplayItems()
let endIsVisible: boolean
let log = new ConsoleLogger()

export let params: {
    userID: string
}

$: userID = UserID.fromString(params.userID)

class DisplayItem {
    item: Item
    userID: string
    signature: string
}


// Whenever we change lazyItems:
$: displayInitialItems(lazyItems)
async function displayInitialItems(lazyItems: AsyncIterator<DisplayItem>) {
    items = []
}

async function displayMoreItems() {
    log.debug("displayMoreItems, endIsVisible", endIsVisible)
    while(endIsVisible) {

        let n = await lazyItems.next()
        if (n.done) {
            log.debug("DisplayMoreItems: no more to display")
            return
        }

        log.debug("showing 1 more item")
        items = [...items, n.value]

        // Wait for Svelte to apply state changes.
        // MAY cause endIsVisibile to toggle off, but at least in Firefox that
        // doesn't always seem to have happened ASAP.
        // I don't mind loading a few more items than necessary, though.
        await tick()
    }
}

async function* getDisplayItems(): AsyncGenerator<DisplayItem> {

    // Prefetch for faster loading:
    let entries = prefetch($appState.client.getUserItems(userID), 4, fetchDisplayItem)

    for await (let item of entries) {
        // We've already logged nulls.
        // TODO: Maybe display some placeholder instead?
        if (item === null) continue

        // For now, we don't display profile updates, because they can be verbose/redundant.
        // TODO: Display some short placeholder that lets people know of a profile update?
        if (item.item.profile) continue
        
        yield item
    }
}

async function fetchDisplayItem(entry: ItemListEntry): Promise<DisplayItem|null> {
    let userID = UserID.fromBytes(entry.user_id.bytes)
    let signature = Signature.fromBytes(entry.signature.bytes)
    let item: Item|null 
    try {
        item = await $appState.client.getItem(userID, signature)
    } catch (e) {
        log.error("Error loading Item:", userID, signature, e)
        return null
    }

    if (item === null) {
        // TODO: Display some placeholder?
        // It does seem like an error, the server told us about the item, but doesn't have it?
        log.error("No such item", userID, signature)
        return null
    }

    return {
        item,
        signature: signature.toString(),
        userID: userID.toString(),
    }
}

</script>