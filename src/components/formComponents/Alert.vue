<template>
    <div v-if="isDisplayed" class="alert alert-danger" role="alert">
        Uwaga nadpłata
    </div>
</template>
<script>
export default {
    name: "Alert",
    props: ['isDisplayed']
}
</script>
