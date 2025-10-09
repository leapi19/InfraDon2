<script setup lang="ts">
 
import { ref } from 'vue'
import PouchDB from 'pouchdb'
//DATA VARIABLES
const counter = ref(0);
 
//FUNCTIONS / METHODS -> CONTROLLER
const increment = () => {
  counter.value++
}

mounted() {
    this.initDatabase()
}

</script>

