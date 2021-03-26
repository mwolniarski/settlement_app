<template>
    <div>
      <div class="alert alert-success" role="alert" v-html="endMessage">
      </div>
      <div>
        <button class="btn btn-warning mb-2 mr-2" v-on:click="$emit('close')">OK</button>
      </div>
    </div>
</template>
<script>
export default {
    name: "EndPanel",
    props:['endMessage']
}
</script>