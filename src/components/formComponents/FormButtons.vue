<template>
     <div>
        <button class="btn btn-success mb-2 mr-2" v-on:click="$emit('settlementDocuments')">OK</button>
        <button class="btn btn-danger mb-2" v-on:click="$emit('close')">Anuluj</button>
    </div>
</template>
