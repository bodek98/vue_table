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
        <th
          v-for="(header, headerIndex) in headers"
          :key="headerIndex"
          @click="sortItems(header)"
          @dragstart="dragStart(headerIndex)"
          @dragover="dragOver(headerIndex)"
          draggable="true"
        >
          {{ header }}
          {{ headerIndex }}
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
      <tr v-for="(row, index) in filteredAndSortedRows" :key="index">
        <td v-for="header in headers" :key="header">
          <template v-if="header === 'COMPONENTS'">
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
          <template v-else-if="header !== 'isExpanded'">
            <td>{{ row[header] }}</td>
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
    const sortState = reactive({
      column: "",
      order: "asc",
    });

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
              (typeof value === "string" || typeof value === "number") &&
              value.toString().toLowerCase().includes(searchText.toLowerCase())
            ) {
              return true;
            }
          }
        }
      }
      return false;
    };

    const deepSearchColumn = (obj, column, searchValue) => {
      for (let key in obj) {
        if (Object.prototype.hasOwnProperty.call(obj, key)) {
          const value = obj[key];
          if (
            (key === column ||
              (column === "S/N" && key === "SN") ||
              (column === "SN" && key === "S/N")) &&
            value.toString().toLowerCase().includes(searchValue.toLowerCase())
          ) {
            return true;
          }
          if (typeof value === "object") {
            if (Array.isArray(value)) {
              for (let i = 0; i < value.length; i++) {
                if (deepSearchColumn(value[i], column, searchValue)) {
                  return true;
                }
              }
            } else {
              if (deepSearchColumn(value, column, searchValue)) {
                return true;
              }
            }
          }
        }
      }
      return false;
    };

    const filteredRows = computed(() => {
      let data = rows.value;
      const searchText = search.value.trim();

      if (searchText.length > 0) {
        return data.filter((row) => {
          return deepSearch(row, searchText);
        });
      }

      if (filters.ID.trim().length > 0) {
        return data.filter((row) => {
          return deepSearchColumn(row, "ID", filters.ID.trim());
        });
      } else if (filters.PRODUCENT.trim().length > 0) {
        return data.filter((row) => {
          return deepSearchColumn(row, "PRODUCENT", filters.PRODUCENT.trim());
        });
      } else if (filters.MODEL.trim().length > 0) {
        return data.filter((row) => {
          return deepSearchColumn(row, "MODEL", filters.MODEL.trim());
        });
      } else if (filters["S/N"].trim().length > 0) {
        return data.filter((row) => {
          return deepSearchColumn(row, "S/N", filters["S/N"].trim());
        });
      }

      return data;
    });

    const filteredAndSortedRows = computed(() => {
      let data = filteredRows.value;

      if (sortState.column && sortState.order) {
        const order = sortState.order === "asc" ? 1 : -1;
        data.value = data.sort((rowA, rowB) => {
          const valueA = rowA[sortState.column];
          const valueB = rowB[sortState.column];
          if (typeof valueA === "string" && typeof valueB === "string") {
            return order * valueA.localeCompare(valueB);
          } else if (typeof valueA === "number" && typeof valueB === "number") {
            return order * (valueA - valueB);
          } else {
            return order * valueA.toString().localeCompare(valueB.toString());
          }
        });
      }

      return data;
    });

    const sortItems = (column) => {
      if (sortState.column === column) {
        sortState.order = sortState.order === "asc" ? "desc" : "asc";
      } else {
        sortState.column = column;
        sortState.order = "asc";
      }
    };

    let dragIndex = null;

    const dragStart = (index) => {
      dragIndex = index;
    };

    const dragOver = (index) => {
      if (dragIndex !== null && dragIndex !== index) {
        const headersCopy = [...headers.value];
        const dragElement = headersCopy.splice(dragIndex, 1)[0];
        headersCopy.splice(index, 0, dragElement);
        headers.value = headersCopy;
        dragIndex = index;
      }
    };

    return {
      headers,
      isExpanded,
      expandRow,
      sortItems,
      filteredAndSortedRows,
      search,
      filters,
      dragStart,
      dragOver,
    };
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped></style>
