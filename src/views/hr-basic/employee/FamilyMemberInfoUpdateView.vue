<template>
  <FlexItem class="content-header" fld="row" h="6rem" w="96%">
    <CommonArticle label="가족" class="ca" w="100%" fs="2rem"></CommonArticle>
    <div class="btns">
      <ButtonItem
        h="3.6rem"
        w="12rem"
        bgc="#003566"
        br="0.6rem"
        c="#fff"
        :fs="'1.6rem'"
        @click="deleteSelectedRows"
      >
        <img src="../../../assets/icons/minus_icon.png" />
        <p>선택 삭제</p></ButtonItem
      >
      <ButtonItem
        h="3.6rem"
        w="12rem"
        bgc="#003566"
        br="0.6rem"
        c="#fff"
        :fs="'1.6rem'"
        @click="addRow"
      >
        <img src="../../../assets/icons/plus_icon.png" />
        <p>행 추가</p></ButtonItem
      >
      <ButtonItem
        h="3.6rem"
        w="12rem"
        bgc="#003566"
        br="0.6rem"
        c="#fff"
        :fs="'1.6rem'"
        @click="postData"
        >수정 요청</ButtonItem
      >
    </div>
  </FlexItem>
  <FlexItem class="content-body" fld="column" h="calc(100% - 3rem)" w="96%">
    <div class="table-wrapper">
      <TableItem class="commute-table" gtc="0.3fr 2fr 1fr 1fr" br="0.5rem">
        <TableRow>
          <TableCell th fs="1.6rem">
            <div class="chk">
              <input
                type="checkbox"
                id="headerCheckbox"
                v-model="headerCheckbox"
                @change="toggleAllCheckboxes"
              />
              <label for="headerCheckbox"></label>
            </div>
          </TableCell>
          <TableCell th fs="1.6rem">성명</TableCell>
          <TableCell th fs="1.6rem">관계</TableCell>
          <TableCell th fs="1.6rem">생년월일</TableCell>
        </TableRow>
        <TableRow
          v-if="!isEmpty"
          v-for="(item, index) in familyList"
          :key="index"
        >
          <TableCell class="mid" fs="1.6rem">
            <div class="chk">
              <input
                type="checkbox"
                :id="'check' + index"
                v-model="selectedRows[index]"
              />
              <label :for="'check' + index"></label>
            </div>
          </TableCell>
          <TableCell
            class="mid"
            v-for="(value, header) in item"
            key="header"
            fs="1.6rem"
          >
            <input
              id="cell-input"
              type="text"
              v-model="familyList[index][header]"
              :class="{
                'invalid-row': !isCellValid(familyList[index][header], header),
              }"
              class="cell-input"
              @focus="showModal(index, header)"
              @blur="hideModal"
            />
            <div
              v-if="visible && activeRow === index && activeHeader === header"
              class="modal"
            >
              <pre>{{ modalTxt[header] }}</pre>
            </div>
          </TableCell>
        </TableRow>
      </TableItem>
    </div>
    <FlexItem
      v-if="isEmpty"
      class="empty-message"
      fld="row"
      h="6rem"
      w="100%"
      fs="1.6rem"
    >
      가족 정보가 존재하지 않습니다.
    </FlexItem>
  </FlexItem>
</template>

<script setup>
import CommonArticle from '@/components/common/CommonArticle.vue';
import ButtonItem from '@/components/semantic/ButtonItem.vue';
import FlexItem from '@/components/semantic/FlexItem.vue';
import TableItem from '@/components/semantic/TableItem.vue';
import TableRow from '@/components/semantic/TableRow.vue';
import TableCell from '@/components/semantic/TableCell.vue';
import { updateData, getRelationships, getFamilyById } from '@/api/emp_attach';
import { ref, onMounted, watch } from 'vue';
import { useRoute, useRouter } from 'vue-router';

const familyList = ref([]);
const isEmpty = ref(true);
const headerCheckbox = ref(false);
const selectedRows = ref([]);
const relationships = ref([]);
const memList = ref([]);

const route = useRoute();
const router = useRouter();

const empId = ref('');

const isCellValid = (value, header) => {
  if (value === null || value === undefined) return false;
  if (typeof value === 'string' && value.trim() === '') return false;
  if (header === 'family_relationship_name')
    return memList.value?.includes(`${value}`);
  if (header === 'birth_date') {
    return /^\d{4}-\d{2}-\d{2}$/.test(value);
  }
  return String(value).length > 0;
};

const visible = ref(false);
const activeRow = ref(null);
const activeHeader = ref(null);

const modalTxt = ref({});

const setModalTxt = () => {
  return {
    birth_date: '입력 예:\n- YYYY-MM-DD',
    family_relationship_name: `선택:\n- ${(memList.value || []).join('\n- ')}`,
  };
};

const showModal = (rowIndex, header) => {
  if (modalTxt.value[header] !== undefined) {
    visible.value = true;
    activeRow.value = rowIndex;
    activeHeader.value = header;
  }
};

const hideModal = () => {
  visible.value = false;
  activeRow.value = null;
  activeHeader.value = null;
};

const sortByDate = (list) => {
  return list.sort((a, b) => {
    const dateA = new Date(a['birth_date']);
    const dateB = new Date(b['birth_date']);
    return dateA - dateB;
  });
};

const original = ref([]);

const fetchDate = async () => {
  const response = await getFamilyById(empId.value);
  const tmp = await getRelationships();
  original.value = response;

  relationships.value = tmp.reduce((acc, row) => {
    acc[row.family_relationship_name] = row.family_relationship_code;
    acc[row.family_relationship_code] = row.family_relationship_name;
    return acc;
  }, {});

  memList.value = tmp.map((row) => row.family_relationship_name);
  if (response) {
    const sortedResponse = sortByDate(
      response.map((row) => ({
        name: row['name'],
        family_relationship_name: row['family_relationship_name'],
        birth_date: row['birth_date'].split('T')[0],
      }))
    );
    familyList.value = sortedResponse;
  } else {
    familyList.value = [];
    isEmpty.value = true;
  }
  isEmpty.value = familyList.value.length === 0;
  initializeSelectedRows();
};

watch(familyList, () => {
  isEmpty.value = familyList.value.length === 0;
  initializeSelectedRows();
});

const initializeSelectedRows = () => {
  selectedRows.value = familyList.value.map(() => false);
  headerCheckbox.value = false;
};

const toggleAllCheckboxes = () => {
  selectedRows.value = selectedRows.value.map(() => headerCheckbox.value);
};

const defaultRow = {
  name: '',
  birth_date: '',
  family_relationship_name: '',
};

const addRow = () => {
  if (!Array.isArray(familyList.value)) {
    familyList.value = [];
  }
  familyList.value.push({ ...defaultRow });
  initializeSelectedRows();
  isEmpty.value = familyList.value.length === 0;
};

const deleteSelectedRows = () => {
  familyList.value = familyList.value.filter(
    (_, index) => !selectedRows.value[index]
  );
  initializeSelectedRows();
};

const mapping = () => {
  return familyList.value.map((row) => {
    const { family_member_id, family_relationship_name, ...rest } = row;
    return {
      ...rest,
      employee_id: empId.value,
      family_relationship_code: relationships.value[family_relationship_name],
    };
  });
};

const postData = async () => {
  const invalidRows = familyList.value.filter((row) => {
    return Object.entries(row).some(
      ([header, value]) => !isCellValid(value, header)
    );
  });

  if (invalidRows.length > 0) {
    window.alert('유효하지 않은 데이터가 있습니다. 데이터를 확인하세요.');
    return;
  }

  const data = mapping();
  try {
    await updateData(
      original.value.map((row) => row.family_member_id),
      data,
      'family-members'
    );
    window.alert('가족 정보 수정 요청이 완료되었습니다.');
    router.push({
      path: '/hr-basic/my-info/familymembers',
      query: {
        employee_id: empId.value,
      },
    });
    return;
  } catch (error) {
    window.alert('수정 요청 중 문제가 발생했습니다. 다시 시도하세요.');
  }
};

onMounted(async () => {
  empId.value = route.query.employee_id || localStorage.getItem('employeeId');
  await fetchDate(empId.value);
  modalTxt.value = setModalTxt();
});
</script>

<style scoped>
.content-header {
  width: 100%;
  position: relative;
  justify-content: space-between;
  align-items: end;
}

.content-body {
  width: 80%;
  margin-top: 1rem;
}

.btns {
  display: flex;
  gap: 1rem;
}

button {
  width: 100px;
  height: 30px;
  flex-shrink: 0;
  border-radius: 5px;
  background: #003566;
  border: none;
  color: #fff;
  font-family: 'Noto Sans KR';
  font-style: normal;
  font-weight: 300;
  line-height: normal;
  align-items: center;
  justify-content: center;
  gap: 1rem;
  cursor: pointer;
  padding: 1px;
}

button p {
  text-align: center;
  vertical-align: middle;
}

.mid {
  justify-content: center;
  align-items: center;
}

.empty-message {
  justify-content: center;
  align-items: center;
}

.TableCell {
  overflow: visible;
  text-align: center;
  vertical-align: middle;
}

.chk {
  display: flex;
  align-items: center;
  justify-content: center;
}

input {
  width: 100%;
  height: 100%;
  padding-left: 0.5rem;
  text-align: left;
  flex-shrink: 0;
  border-radius: 1px;
  border: 0.586px solid #dbdbdb;
  background: #f8f8f8;
  box-shadow: 0px 0.977px 1.954px 0px rgba(0, 0, 0, 0.25) inset;
}
.chbox {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}

.chbox > label {
  display: flex;
  align-items: center;
  justify-content: center;
}

input[type='checkbox'] {
  display: none;
}

input[type='checkbox'] + label {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  border: 1px solid #dbdbdb;
  position: relative;
}

input[type='checkbox']:checked + label::after {
  content: '✔';
  font-size: 20px;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #000;
}

.mid {
  position: relative;
}

.modal {
  display: flex;
  flex-direction: column;
  text-align: left;
  position: absolute;
  top: 100%;
  left: 0.5rem;
  min-width: max-content;
  min-height: max-content;
  padding: 1.5rem;
  margin-top: 0.2rem;
  background-color: #fff;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.1);
  font-size: 9px;
  z-index: 9999;
}

.modal pre {
  margin: 0;
  color: #333;
  font-size: 9px;
  line-height: 1.5;
}

.invalid-row {
  background: #ffd8d8 !important;
  stroke: #f00 !important;
  border: 2px solid red !important;
}
</style>
