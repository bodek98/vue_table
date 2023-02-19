<template>
  <div>
    <input type="text" placeholder="Search" v-model="search" />
  </div>
  <table id="data-table" class="table">
    <div id="expanded-rows-hideout"></div>
    <thead>
      <tr>
        <th
          v-for="(header, headerIndex) in headers"
          :key="headerIndex"
          @dragstart="dragStart(headerIndex)"
          @dragover="dragOver(headerIndex)"
          draggable="true"
        >
          {{ header.title }}
          <template v-if="header.title !== 'COMPONENTS'">
            <button @click="toggleFilter(headerIndex)">
              {{ header.isFilterClicked ? "Hide" : "Show" }}
            </button>
            <button @click="sortItems(header.title)">Sort</button>
            <template v-if="header.isFilterClicked">
              <input
                type="text"
                :placeholder="header.title"
                v-model="filters[header.title]"
              />
            </template>
          </template>
        </th>
      </tr>
    </thead>
    <tbody class="table__body">
      <tr
        class="table__bodyRow"
        v-for="(row, index) in filteredAndSortedRows"
        :key="index"
        :id="'table-data-id-' + row.localID"
      >
        <td class="table__bodyCell" v-for="header in headers" :key="header">
          <template v-if="header.title === 'COMPONENTS'">
            <button @click="expandRow(row)">v</button>
          </template>
          <template v-else-if="header.title !== 'isExpanded'">
            <td>{{ row[header.title] }}</td>
          </template>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script>
import {
  computed,
  onMounted,
  onUpdated,
  reactive,
  ref,
} from "@vue/runtime-core";
import axios from "axios";
export default {
  name: "DataTable",
  props: {
    msg: String,
  },
  setup() {
    const baseUrl = ref("http://localhost:3000");
    let urls = ref([]);
    let headers = ref([
      { title: "ID", isFilterClicked: false },
      { title: "PRODUCENT", isFilterClicked: false },
      { title: "MODEL", isFilterClicked: false },
      { title: "S/N", isFilterClicked: false },
      { title: "COMPONENTS", isFilterClicked: false },
    ]);
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
            localID: i.value++,
          }));
          rows.value = data;
        } catch (error) {
          console.error(error);
        }
      };
      fetchData();
    });

    onUpdated(() => {
      stickExpandedToParents();
    });

    let rowItem = ref(null);
    const expandRow = (row) => {
      let table = document.getElementById("data-table");
      const parentHtmlId = "table-data-id-" + row.localID;
      let htmlRowId = document.querySelector("tr#" + parentHtmlId).rowIndex;
      const components = Object.values(row.COMPONENTS);
      if (row.isExpanded === false) {
        row.isExpanded = true;
        components.forEach((component, index) => {
          let newRow = table.insertRow(htmlRowId + 1 + index);
          newRow.classList.add("expandedRow");
          newRow.setAttribute("parent-id", parentHtmlId);
          for (let i = 0; i < headers.value.length; i++) {
            const header = headers.value[i];
            let headerTitle = header.title;
            if (headerTitle === "S/N") {
              headerTitle = "SN";
            }
            newRow.insertCell(i);
            newRow.cells[i].innerHTML = component[headerTitle]
              ? component[headerTitle]
              : "";
            newRow.cells[i].setAttribute("header-id", headerTitle);
          }
        });
      } else {
        row.isExpanded = false;
        for (let i = 0; i < components.length; i++) {
          table.deleteRow(htmlRowId + 1);
        }
      }
    };

    const toggleFilter = (index) => {
      if (headers.value) {
        headers.value[index].isFilterClicked =
          !headers.value[index].isFilterClicked;
      }
    };

    const deepSearch = (obj, searchText) => {
      const stack = [obj];

      while (stack.length > 0) {
        const currentObj = stack.pop();

        if (typeof currentObj === "object" && currentObj !== null) {
          for (const key in currentObj) {
            if (Object.prototype.hasOwnProperty.call(currentObj, key)) {
              const value = currentObj[key];
              if (typeof value === "object" && value !== null) {
                stack.push(value);
              } else if (
                (typeof value === "string" || typeof value === "number") &&
                value
                  .toString()
                  .toLowerCase()
                  .includes(searchText.toLowerCase())
              ) {
                return true;
              }
            }
          }
        }
      }

      return false;
    };

    const deepSearchColumn = (obj, column, searchValue) => {
      const stack = [obj];

      while (stack.length > 0) {
        const currentObj = stack.pop();

        for (let key in currentObj) {
          if (Object.prototype.hasOwnProperty.call(currentObj, key)) {
            const value = currentObj[key];
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
                  stack.push(value[i]);
                }
              } else {
                stack.push(value);
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

    const cloneAttributes = (target, source) => {
      [...source.attributes].forEach((attr) => {
        target.setAttribute(
          attr.nodeName === "id" ? "data-id" : attr.nodeName,
          attr.nodeValue
        );
      });
    };

    const stickExpandedToParents = () => {
      const expandedRows = document.querySelectorAll(".expandedRow");
      let table = document.getElementById("data-table");
      for (let i = expandedRows.length - 1; i >= 0; i--) {
        const expandedRow = expandedRows[i];
        const parentHtmlId = expandedRow.getAttribute("parent-id");
        let parentElement = document.querySelector("tr#" + parentHtmlId);
        if (!parentElement) {
          document
            .getElementById("expanded-rows-hideout")
            .appendChild(expandedRow);
          continue;
        }
        let htmlRowId = parentElement.rowIndex;
        let newRow = table.insertRow(htmlRowId + 1);
        cloneAttributes(newRow, expandedRow);
        replaceRow(newRow, expandedRow);
        expandedRow.remove();
      }
    };

    const replaceRow = (target, source) => {
      for (let i = 0; i < headers.value.length; i++) {
        const header = headers.value[i];
        let headerTitle = header.title;
        if (headerTitle === "S/N") {
          headerTitle = "SN";
        }
        let oldCell = source.querySelector("[header-id=" + headerTitle + "]");
        if (!oldCell) {
          target.insertCell(i);
          continue;
        }
        target.insertCell(i);
        target.cells[i].innerHTML = oldCell.innerHTML;
        cloneAttributes(target.cells[i], oldCell);
        oldCell.remove();
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
      toggleFilter,
      sortItems,
      filteredAndSortedRows,
      search,
      filters,
      dragStart,
      dragOver,
      rowItem,
    };
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style src="./DataTable.scss" lang="scss"></style>
