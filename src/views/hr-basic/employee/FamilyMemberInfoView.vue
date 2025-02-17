<template>
  <CommonArticle label="가족" class="ca" w="96%" fs="2rem">
    <ButtonItem
      v-if="props.employee_id === undefined"
      class="update-btn"
      h="3.6rem"
      w="7.2rem"
      bgc="#003566"
      br="0.6rem"
      c="#fff"
      :fs="'1.6rem'"
      @click="handleOnclick"
      >수정</ButtonItem
    >
    <FlexItem class="content-body" fld="column" h="calc(100% - 3rem)" w="100%">
      <div class="table-wrapper">
        <TableItem class="commute-table" gtc="0.3fr 2fr 1fr 1fr" br="0.5rem">
          <TableRow>
            <TableCell th fs="1.6rem">No</TableCell>
            <TableCell th fs="1.6rem">성명</TableCell>
            <TableCell th fs="1.6rem">관계</TableCell>
            <TableCell th fs="1.6rem">생년월일</TableCell>
          </TableRow>
          <TableRow
            v-if="!isEmpty"
            v-for="(item, index) in familyList"
            :key="index"
          >
            <TableCell class="mid" fs="1.6rem">{{ index + 1 }}</TableCell>
            <TableCell class="mid" fs="1.6rem">{{ item['name'] }}</TableCell>
            <TableCell class="mid" fs="1.6rem">{{
              item['family_relationship_name']
            }}</TableCell>
            <TableCell class="mid" fs="1.6rem">{{
              item['birth_date']
            }}</TableCell>
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
  </CommonArticle>
</template>

<script setup>
import CommonArticle from '@/components/common/CommonArticle.vue';
import ButtonItem from '@/components/semantic/ButtonItem.vue';
import FlexItem from '@/components/semantic/FlexItem.vue';
import TableItem from '@/components/semantic/TableItem.vue';
import TableRow from '@/components/semantic/TableRow.vue';
import TableCell from '@/components/semantic/TableCell.vue';
import { getFamilyById } from '@/api/emp_attach';
import { ref, onMounted, watch } from 'vue';
import { useRouter, useRoute } from 'vue-router';

const familyList = ref([]);
const isEmpty = ref(true);

const router = useRouter();
const route = useRoute();
const employeeId = ref('');

const props = defineProps({
  employee_id: {
    type: String,
    required: false,
  },
});

const sortByDate = (list) => {
  return list.sort((a, b) => {
    const dateA = new Date(a['birth_date']);
    const dateB = new Date(b['birth_date']);
    return dateA - dateB;
  });
};

const fetchDate = async (empId) => {
  if (!empId) return;
  const response = await getFamilyById(empId);
  if (response) {
    const sortedResponse = sortByDate(response.map((row) => ({
      ...row,
      birth_date: row['birth_date'].split('T')[0],
    })));
    familyList.value = sortedResponse;
    isEmpty.value = familyList.value.length === 0;
  } else {
    familyList.value = [];
    isEmpty.value = true;
  }
};

const handleOnclick = () => {
  router.push({
    path: '/hr-basic/my-info/familymembers/update',
    query: {
      employee_id: employeeId.value,
    },
  });
  return;
};

onMounted(() => {
  employeeId.value =
    route.query.employee_id ||
    props.employee_id ||
    localStorage.getItem('employeeId');
});

watch(
  [() => route.query.employee_id, () => props.employee_id],
  ([queryId, propId], [oldQueryId, oldPropId]) => {
    const newId = queryId || propId || localStorage.getItem('employeeId');
    if (newId !== employeeId.value) {
      employeeId.value = newId;
      fetchDate(newId);
    }
  },
  { immediate: true }
);
</script>

<style scoped>
.common-article {
  position: relative;
  margin-top: 2rem;
}

.ca {
  align-self: center;
}

.update-btn {
  position: absolute;
  top: 0;
  right: 0;
}

.content-body {
  width: 80%;
  margin-top: 1rem;
}

.mid {
  justify-content: center;
  align-items: center;
}

.empty-message {
  justify-content: center;
  align-items: center;
}
</style>
