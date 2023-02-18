<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
  </div>
  <div>
    <input type="text" placeholder="Search" v-model="search" />
    {{ search }}
  </div>
  <table>
    <thead>
      <tr>
        <th v-for="(header, index) in headers" :key="index">
          {{ header }}
          <template v-if="header !== 'COMPONENTS'">
            <input
              type="text"
              :placeholder="header"
              v-model="filters[header]"
            />
          </template>
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(row, index) in filteredRows" :key="index">
        <td v-for="(rowItem, itemKey, idx) in row" :key="idx">
          <template v-if="itemKey === 'COMPONENTS'">
            <button @click="expandRow(row)">v</button>
            <template v-if="row.isExpanded">
              <tr v-for="(component, cIndex) in row.COMPONENTS" :key="cIndex">
                <td>{{ component.ID }}</td>
                <td>{{ component.PRODUCENT }}</td>
                <td>{{ component.MODEL }}</td>
                <td>{{ component.SN }}</td>
              </tr>
            </template>
          </template>
          <template v-else-if="itemKey !== 'isExpanded'">
            <td>{{ rowItem }}</td>
          </template>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script>
import { computed, onMounted, reactive, ref } from "@vue/runtime-core";
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
    let search = ref("");
    const filters = reactive({ ID: "", PRODUCENT: "", MODEL: "", "S/N": "" });
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
      if (row.isExpanded === false) {
        row.isExpanded = true;
      } else {
        row.isExpanded = false;
      }
    };

    const deepSearch = (obj, searchText) => {
      if (typeof obj === "object" && obj !== null) {
        for (const key in obj) {
          if (Object.prototype.hasOwnProperty.call(obj, key)) {
            const value = obj[key];
            if (typeof value === "object" && value !== null) {
              if (deepSearch(value, searchText)) {
                return true;
              }
            } else if (
              typeof value === "string" &&
              value.toLowerCase().includes(searchText.toLowerCase())
            ) {
              return true;
            }
          }
        }
      }
      return false;
    };

    const filteredRows = computed(() => {
      const searchText = search.value.trim();

      if (searchText.length > 0) {
        return rows.value.filter((row) => {
          return deepSearch(row, searchText);
        });
      }
      if (filters.ID.trim().length > 0) {
        return rows.value.filter((row) =>
          row.ID.toString().toLowerCase().includes(filters.ID.toLowerCase())
        );
      }
      if (filters.PRODUCENT.trim().length > 0) {
        return rows.value.filter((row) =>
          row.PRODUCENT.toLowerCase().includes(filters.PRODUCENT.toLowerCase())
        );
      }
      if (filters.MODEL.trim().length > 0) {
        return rows.value.filter((row) =>
          row.MODEL.toLowerCase().includes(filters.MODEL.toLowerCase())
        );
      }
      if (filters["S/N"].trim().length > 0) {
        return rows.value.filter((row) =>
          row["S/N"].toLowerCase().includes(filters["S/N"].toLowerCase())
        );
      }
      return rows.value;
    });

    return {
      rows,
      headers,
      isExpanded,
      expandRow,
      filteredRows,
      search,
      filters,
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
