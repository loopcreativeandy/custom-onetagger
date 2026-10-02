<template>

<div class='q-pa-md'>
    <!-- Note -->
    <div class='full-width row justify-center q-my-xs'>
            <q-btn outline color='primary' class='text-caption' v-if='$1t.quickTag.value.track' @click='$1t.onQuickTagEvent("onNoteTag")'>         
            Add custom note            
        </q-btn>
    </div>
    <!-- Tag filter: type to filter, Up/Down to move, Enter to toggle, Esc to clear -->
    <div class='tag-filter-sticky'>
        <q-input
            ref='filterRef'
            v-model='filter'
            dense
            filled
            clearable
            label='Filter tags (Ctrl+F)'
            @update:model-value='highlight = 0'
            @keydown='filterKeydown'
        ></q-input>
    </div>

    <div v-for='(tag, i) in $1t.settings.value.quickTag.custom' :key='"tag"+i' class='q-pb-md' v-show='!filtering || matchesByTag[i]'>
        <!-- Tag title -->
        <q-expansion-item 
            :label='tag.name' 
            class='text-subtitle2 text-bold q-pb-sm'
            style='margin-bottom: -24px;'
            default-opened
            :model-value="true"
            :switch-toggle-side='false'
        >
            <!-- Values -->
            <div
                v-for='(value, j) in tag.values'
                :key='i+"value"+j'
                v-show='!filtering || isMatch(i, j)'
                :class='{"tag-filter-highlight": filtering && isHighlighted(i, j)}'
            >
                <q-checkbox
                    :label='value.val'
                    :model-value='selected(i, value.val)'
                    @update:model-value='valueClick(i, value.val)'
                    dense
                    class='text-subtitle2 text-grey-5 full-width q-mt-xs'
                ></q-checkbox>
            </div>

            <!-- Add new -->
            <q-input ref='addNewTagRef' dense @keypress.enter="addNewTag" v-if='newTag == i' v-model='newTagValue'></q-input>

            <q-btn round flat color='primary' class='add-custom-btn' v-if='newTag == -1' @click='showNewTag(i)'>
                <q-icon name='mdi-plus'></q-icon>
            </q-btn>

        </q-expansion-item>

        <div class='q-mb-md'></div>
    </div>

    <!-- Reorder the values inside of tag -->
    <!-- <div class='full-width row justify-center'>
        <q-btn outline color='primary' class='text-caption'  @click='sortValues' v-if='$1t.quickTag.value.track'>Sort values</q-btn>
    </div> -->

    <!-- Manual tag -->
    <div class='full-width row justify-center' v-if='$1t.quickTag.value.track.tracks.length == 1'>
        <q-btn outline color='primary' class='text-caption' @click='$1t.onQuickTagEvent("onManualTag", {path: $1t.quickTag.value.track.tracks[0].path})'>MANUAL TAG</q-btn>
    </div>

</div>
</template>

<script lang='ts' setup>
import { computed, nextTick, onMounted, onUnmounted, ref } from 'vue';
import { get1t } from '../scripts/onetagger.js';

const $1t = get1t();
const newTag = ref(-1);
const newTagValue = ref<string | undefined>(undefined);

// Tag filter
const filter = ref<string | null>(null);
const highlight = ref(0);
const filterRef = ref<any>();

const filterTokens = computed(() => (filter.value ?? '').toLowerCase().split(/\s+/).filter(t => t.length > 0));
const filtering = computed(() => filterTokens.value.length > 0);

// Every query word must appear in the value or its category name
const matches = computed(() => {
    let out: { tag: number, value: number, val: string }[] = [];
    if (!filtering.value) return out;
    $1t.settings.value.quickTag.custom.forEach((tag, i) => {
        tag.values.forEach((value, j) => {
            let haystack = `${value.val} ${tag.name}`.toLowerCase();
            if (filterTokens.value.every(t => haystack.includes(t)))
                out.push({ tag: i, value: j, val: value.val });
        });
    });
    // Matches on the value itself first, then prefix matches
    const score = (m: { val: string }) => {
        let v = m.val.toLowerCase();
        if (!filterTokens.value.every(t => v.includes(t))) return 2;
        return v.startsWith(filterTokens.value[0]) ? 0 : 1;
    };
    return out.map((m, k) => ({ m, k, s: score(m) })).sort((a, b) => a.s - b.s || a.k - b.k).map(x => x.m);
});
const matchesByTag = computed(() => {
    let out: Record<number, boolean> = {};
    matches.value.forEach(m => out[m.tag] = true);
    return out;
});

function isMatch(tag: number, value: number) {
    return matches.value.some(m => m.tag == tag && m.value == value);
}
function isHighlighted(tag: number, value: number) {
    let m = matches.value[highlight.value];
    return m && m.tag == tag && m.value == value;
}

function filterKeydown(e: KeyboardEvent) {
    if (e.key == 'ArrowDown' || e.key == 'ArrowUp') {
        let n = matches.value.length;
        if (n > 0) highlight.value = (highlight.value + (e.key == 'ArrowDown' ? 1 : n - 1)) % n;
        e.preventDefault();
        nextTick(() => document.querySelector('.tag-filter-highlight')?.scrollIntoView({ block: 'nearest' }));
        return;
    }
    if (e.key == 'Enter') {
        e.preventDefault();
        let m = matches.value[highlight.value];
        // Empty filter or no match: leave the field so keybinds work again
        if (!m) {
            filterRef.value?.blur();
            return;
        }
        if ($1t.quickTag.value.track.hasTracks())
            valueClick(m.tag, $1t.settings.value.quickTag.custom[m.tag].values[m.value].val);
        // Clear and leave the field so track keys (Up/Down, Space) work straight away;
        // Ctrl+F again for the next tag
        filter.value = null;
        highlight.value = 0;
        filterRef.value?.blur();
        return;
    }
    if (e.key == 'Escape') {
        e.preventDefault();
        filter.value = null;
        highlight.value = 0;
        filterRef.value?.blur();
    }
}

function focusFilter() {
    filterRef.value?.focus();
    filterRef.value?.select();
}
onMounted(() => window.addEventListener('1t-focus-tag-filter', focusFilter));
onUnmounted(() => window.removeEventListener('1t-focus-tag-filter', focusFilter));

// If the value is present in tag
function selected(tag: number, value: string) {
    return $1t.quickTag.value.track.getCustom(tag, value);
}

// Tag value click
function valueClick(tag: number, value: string) {
    $1t.quickTag.value.track.toggleCustom(tag, value);           
}

// Sort values inside of tag
function sortValues() {
    $1t.quickTag.value.track.sortCustom();
}

// Show add new tag input
const addNewTagRef = ref<any>();
function showNewTag(i: number) {
    newTag.value = i;
    setTimeout(() => {
        addNewTagRef.value[0].focus();
    }, 25);
}

// Adding new tag
function addNewTag() {
    if (newTagValue.value) {
        $1t.settings.value.quickTag.custom[newTag.value].values.push({val: newTagValue.value, keybind: undefined});
        $1t.saveSettings();
    }
    newTag.value = -1;
    newTagValue.value = undefined;
}

</script>

<style>
.q-expansion-item__container:first-child div {
    padding: 0px !important;
}
.q-checkbox__label {
    margin-left: 8px !important;
}
.add-custom-btn {
    position: absolute;
    bottom: -10px;
    left: 128px;
}
.tag-filter-sticky {
    position: sticky;
    top: 0;
    z-index: 10;
    background: #202020;
    margin: -16px -16px 16px -16px;
    padding: 16px 16px 0 16px;
}
.tag-filter-highlight {
    background: rgba(0, 210, 191, 0.2);
    border-radius: 4px;
}
.hide-expand-icon {
    display: none !important;
}
</style>