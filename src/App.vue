<script setup>
import ScratchCard from './components/ScratchCard.vue'
</script>

<template>
  <div class="project">
    <div class="warp">
      <ScratchCard />
    </div>
  </div>
</template>

<style scoped lang="scss">
.project {
  min-height: 100vh;
  padding: 150px 0;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  // .warp {
  //   max-width: 800px;
  //   margin: 0 auto;
  //   border-radius: 20px;
  //   background-color: #3bd3a5;
  //   padding: 60px 30px;
  // }
}
</style>
