<script setup lang="ts">
import {ref} from 'vue';
//DATA 
const counter = ref(0);

//FUNCTION
const increment = ()=> {
  counter.value++;
}
</script>

<template>
  <h1>Welcome</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">Increment</button>
</template>
