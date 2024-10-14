<template>
  <q-page class="row q-pt-xl">
    <div class="full-width q-px-xl">
      <div class="q-mb-xl">
        <div class="q-mb-md">
          <q-input v-model="searchQuery" label="搜索" dense>
            <template v-slot:append>
              <q-icon name="search" />
            </template>
          </q-input>
        </div>
        <q-form @submit="onSubmit" @reset="resetForm" class="q-gutter-md">
          <q-input
            v-model="tempData.name"
            label="姓名"
            :rules="[(val) => !!val || '姓名不得空白']"
          />
          <q-input
            v-model.number="tempData.age"
            label="年齡"
            type="number"
            :rules="[
              (val) => !!val || '年齡不得空白',
              (val) => val > 0 || '年齡必須為正整數',
            ]"
          />
          <q-btn type="reset" label="清除" color="black" class="q-mt-md" />
          <q-btn
            type="submit"
            :label="isEditing ? '更新' : '新增'"
            :color="isEditing ? 'orange' : 'primary'"
            class="q-mt-md"
          />
        </q-form>
      </div>

      <q-table
        flat
        bordered
        ref="tableRef"
        :rows="filteredData"
        :columns="(tableConfig as QTableProps['columns'])"
        row-key="id"
        :pagination="pagination"
        @request="onRequest"
        :loading="loading"
      >
        <template v-slot:body="props">
          <q-tr :props="props">
            <q-td v-for="col in props.cols" :key="col.name" :props="props">
              {{ col.value }}
            </q-td>
            <q-td auto-width>
              <q-btn
                v-for="btn in tableButtons"
                :key="btn.status"
                size="sm"
                color="grey-6"
                round
                dense
                :icon="btn.icon"
                class="q-ml-sm"
                @click="handleClickOption(btn, props.row)"
              >
                <q-tooltip>{{ btn.label }}</q-tooltip>
              </q-btn>
            </q-td>
          </q-tr>
        </template>
      </q-table>
    </div>

    <q-dialog v-model="confirmDelete" persistent>
      <q-card>
        <q-card-section class="row items-center">
          <span class="q-ml-sm">確定要刪除這筆資料嗎？</span>
        </q-card-section>
        <q-card-actions align="right">
          <q-btn flat label="取消" color="primary" v-close-popup />
          <q-btn
            flat
            label="確定"
            color="primary"
            @click="confirmDeleteAction"
            v-close-popup
          />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';
import { QTableProps } from 'quasar';
import { useQuasar } from 'quasar';

const $q = useQuasar();

const blockData = ref([]);
const loading = ref(false);
const isEditing = ref(false);
const confirmDelete = ref(false);
const deleteId = ref(null);
const searchQuery = ref('');

const pagination = ref({
  sortBy: 'name',
  descending: false,
  page: 1,
  rowsPerPage: 10,
  rowsNumber: 0,
});

const tableConfig = [
  {
    label: '姓名',
    name: 'name',
    field: 'name',
    align: 'left',
    required: true,
    sortable: true,
  },
  { label: '年齡', name: 'age', field: 'age', align: 'left', sortable: true },
];

const tableButtons = [
  { label: '編輯', icon: 'edit', status: 'edit' },
  { label: '刪除', icon: 'delete', status: 'delete' },
];

const tempData = ref({
  name: '',
  age: '',
});

const apiUrl = 'https://dahua.metcfire.com.tw/api/CRUDTest';

onMounted(() => {
  fetchData();
});

async function fetchData() {
  loading.value = true;
  try {
    const response = await axios.get(apiUrl + '/a');
    blockData.value = response.data;
    pagination.value.rowsNumber = response.data.length;
  } catch (error) {
    console.error('Error fetching data:', error);
  } finally {
    loading.value = false;
  }
}

const filteredData = computed(() => {
  const query = searchQuery.value.toString();
  return blockData.value.filter(
    (person) =>
      person.name.toString().includes(query) ||
      person.age.toString().includes(query)
  );
});

async function onSubmit() {
  if (isEditing.value) {
    await updatePerson();
  } else {
    await addPerson();
  }
}

async function addPerson() {
  try {
    await axios.post(apiUrl, tempData.value);
    await fetchData();
    resetForm();
  } catch (error) {
    console.error('Error adding person:', error);
  }
}

async function updatePerson() {
  try {
    await axios.patch(`${apiUrl}`, tempData.value);
    await fetchData();
    resetForm();
  } catch (error) {
    console.error('Error updating person:', error);
  }
}

function resetForm() {
  tempData.value = { name: '', age: 0 };
  isEditing.value = false;
}

function handleClickOption(btn, row) {
  if (btn.status === 'edit') {
    tempData.value = { ...row };
    isEditing.value = true;
  } else if (btn.status === 'delete') {
    deleteId.value = row.id;
    confirmDelete.value = true;
  }
}

async function confirmDeleteAction() {
  if (deleteId.value) {
    try {
      await axios.delete(`${apiUrl}/${deleteId.value}`);
      await fetchData();
    } catch (error) {
      console.error('Error deleting person:', error);
    }
  }
}

function onRequest(props: { pagination: typeof pagination.value }) {
  const { page, rowsPerPage, sortBy, descending } = props.pagination;
  pagination.value = { ...pagination.value, ...props.pagination };

  const sortedData = [...filteredData.value].sort((a, b) => {
    const modifier = descending ? -1 : 1;
    if (a[sortBy] < b[sortBy]) return -1 * modifier;
    if (a[sortBy] > b[sortBy]) return 1 * modifier;
    return 0;
  });

  const startIndex = (page - 1) * rowsPerPage;
  return sortedData.slice(startIndex, startIndex + rowsPerPage);
}
</script>

<style lang="scss" scoped>
.q-table th {
  font-size: 20px;
  font-weight: bold;
}

.q-table tbody td {
  font-size: 18px;
}
</style>
