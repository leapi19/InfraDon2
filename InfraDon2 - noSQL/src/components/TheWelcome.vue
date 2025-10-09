<script setup lang="ts">
 
import { ref } from 'vue'
import PouchDB from 'pouchdb
//DATA VARIABLES
const counter = ref(0);
 
//FUNCTIONS / METHODS -> CONTROLLER
const increment = () => {
  counter.value++
}

const db = new PouchDB(http)

 
</script>

mounted() {
    this.initDatabase()
}