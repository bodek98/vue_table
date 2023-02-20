<template>
  <DataTable msg="Welcome to Your Vue.js App" />
</template>

<script>
import DataTable from "./components/DataTable.vue";

export default {
  name: "App",
  components: {
    DataTable,
  },
};
</script>

<style>
@import url("https://fonts.googleapis.com/css2?family=Noto+Sans:wght@300;400;500;600&display=swap");
* {
  box-sizing: border-box;
}
body {
  font-family: "Poppins", sans-serif;
  font-size: 1.5rem;
  background-color: #222222;
}
#app {
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 50px;
  display: flex;
  flex-direction: column;
  align-items: center;
  font-size: 1.5rem;
}
</style>
