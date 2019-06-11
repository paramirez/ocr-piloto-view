<template>
  <a-layout id="app">
    <a-layout-content :style="{ padding: '0 50px'}">
      <div :style="{ background: '#fff', padding: '24px', minHeight: '380px' }">
        <Ocr/>
      </div>
    </a-layout-content>
  </a-layout>
</template>

<script>
import Ocr from "./components/Ocr.vue";

export default {
  name: "app",
  components: {
    Ocr
  }
};
</script>
