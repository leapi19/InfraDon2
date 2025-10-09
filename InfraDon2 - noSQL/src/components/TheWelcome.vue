<script setup lang="ts">
 
import { ref } from 'vue'
//DATA VARIABLES
const counter = ref(0);
 
//FUNCTIONS / METHODS -> CONTROLLER
const increment = () => {
  counter.value++
}
 
</script>