<template>
  <CommonArticle label="어학" class="ca" w="96%" fs="2rem">
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
        <TableItem
          class="commute-table"
          gtc="0.5fr 1fr 1.5fr 1fr 1fr 1.5fr 1.5fr"
          br="0.5rem"
        >
          <TableRow>
            <TableCell th fs="1.6rem">No</TableCell>
            <TableCell th fs="1.6rem">언어</TableCell>
            <TableCell th fs="1.6rem">어학 자격증</TableCell>
            <TableCell th fs="1.6rem">자격번호</TableCell>
            <TableCell th fs="1.6rem">취득일</TableCell>
            <TableCell th fs="1.6rem">발급기관</TableCell>
            <TableCell th fs="1.6rem">등급 및 점수</TableCell>
          </TableRow>
          <TableRow
            v-if="!isEmpty"
            v-for="(item, index) in langTestList"
            :key="index"
          >
            <TableCell class="mid" fs="1.6rem">{{ index + 1 }}</TableCell>
            <TableCell class="mid" fs="1.6rem">{{
              item['language_name']
            }}</TableCell>
            <TableCell class="mid" fs="1.6rem">{{
              item['language_test_name']
            }}</TableCell>
            <TableCell class="mid" fs="1.6rem">{{
              item['qualification_number']
            }}</TableCell>
            <TableCell class="mid" fs="1.6rem">{{
              item['qualified_at']
            }}</TableCell>
            <TableCell class="mid" fs="1.6rem">{{ item['issuer'] }}</TableCell>
            <TableCell class="mid" fs="1.6rem">{{
              item['grade_score']
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
        어학 자격증 정보가 존재하지 않습니다.
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
import { getLanguageTestsById } from '@/api/emp_attach';
import { ref, onMounted, watch } from 'vue';
import { useRouter, useRoute } from 'vue-router';

const langTestList = ref([]);
const isEmpty = ref(true);

const router = useRouter();
const route = useRoute();
const langCodes = ref([]);

const employeeId = ref('');

const props = defineProps({
  employee_id: {
    type: String,
    required: false,
  },
});

const sortByDate = (list) => {
  return list.sort((a, b) => {
    const dateA = new Date(a['qualified_at']);
    const dateB = new Date(b['qualified_at']);
    return dateA - dateB;
  });
};

const fetchDate = async (empId) => {
  if (!empId) return;
  const response = await getLanguageTestsById(empId);
  if (response) {
    const sortedResponse = sortByDate(response);
    langTestList.value = sortedResponse;
    isEmpty.value = langTestList.value.length === 0;
  } else {
    langTestList.value = [];
    isEmpty.value = true;
  }
};

const handleOnclick = () => {
  router.push({
    path: '/hr-basic/my-info/languagetests/update',
    query: {
      employee_id: employeeId.value,
    },
  });
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
