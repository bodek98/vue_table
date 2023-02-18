<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
  </div>
  <table>
    <thead>
      <tr>
        <th v-for="(header, index) in headers" :key="index">{{ header }}</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(row, index) in rows" :key="index">
        <td v-for="(rowItem, itemKey, idx) in row" :key="idx">
          <template v-if="idx === rows.length - 3">
            <button @click="expandRow(row, rowItem)">Show</button>
            <template v-if="row.isExpanded">
              <tr>
                <td>{{ row.COMPONENTS }}</td>
              </tr>
            </template>
          </template>
          <template v-else> {{ rowItem }} </template>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script>
import { onMounted, ref } from "@vue/runtime-core";
import axios from "axios";
export default {
  name: "DataTable",
  props: {
    msg: String,
  },
  setup() {
    const baseUrl = ref("http://localhost:3000");
    let urls = ref([]);
    const headers = ref(["ID", "PRODUCENT", "MODEL", "S/N", "COMPONENTS"]);
    let rows = ref([]);
    const isExpanded = ref(false);
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
          const data = responses.map((response) => ({
            ...response.data,
            isExpanded: false,
          }));
          rows.value = data;
        } catch (error) {
          console.error(error);
        }
        console.log(rows.value[1].COMPONENTS);
      };
      fetchData();
    });
    const expandRow = (row) => {
      row.isExpanded = true;
    };
    return {
      rows,
      headers,
      isExpanded,
      expandRow,
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
