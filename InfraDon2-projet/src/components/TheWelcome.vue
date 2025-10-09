<script setup lang="ts">
import {ref} from 'vue';
import PouchDB from 'pouchdb';
//DATA 
const counter = ref(0);

//FUNCTION
const increment = ()=> {
  counter.value++;
}

mounted(){
    this.initDatabase()
    this.fetchData()
}

const db = new PouchDB('http://admin:admin@localhost:5984/infra_53_0850')

</script>

<template>
  <h1>Welcome</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">Increment</button>
</template>


