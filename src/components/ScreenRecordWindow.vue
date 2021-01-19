<template>
  <div class="screen-record-window">

  </div>
</template>

<script>
export default {
  name: 'ScreenRecordWindow',
  props: {
    msg: String
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.screen-record-window {
  border:1px solid #ccc;
  width: 800px;
  height: 400px;
  margin: 30px auto;
  display:none;
}
</style>
