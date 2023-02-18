<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
  </div>
  <table>
    <thead>
      <tr>
        <th>column</th>
        <th>column</th>
        <th>column</th>
        <th>column</th>
        <th>column</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(item, index) in items" :key="index">
        <td>{{ index }}</td>
        <td>{{ item.ID }}</td>
        <td>{{ item.PRODUCENT }}</td>
        <td>{{ item.MODEL }}</td>
        <td>{{ item["S/N"] }}</td>
        <!-- <td>{{ item.COMPONENTS }}</td> -->
      </tr>
    </tbody>
  </table>
</template>

<script>
import { onMounted, ref } from "@vue/runtime-core";
import axios from "axios";
export default {
  name: "HelloWorld",
  props: {
    msg: String,
  },
  setup() {
    const baseUrl = ref("http://localhost:3000");
    let urls = ref([]);
    let items = ref([]);
    onMounted(async () => {
      let i = ref(0);
      let keepGoing = true;
      while (keepGoing) {
        const url = `${baseUrl.value}/${i.value}`;
        try {
          await axios.head(url);
          urls.value.push(url);
          i.value++;
        } catch (error) {
          keepGoing = false;
        }
      }
      const fetchData = async () => {
        try {
          const responses = await Promise.all(
            urls.value.map((url) => axios.get(url))
          );
          const data = responses.map((response) => response.data);
          items.value = data;
        } catch (error) {
          console.error(error);
        }
      };
      fetchData();
    });
    return {
      items,
    };
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
