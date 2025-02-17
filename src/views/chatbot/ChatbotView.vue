<template>
  <CommonNav :cur="2"></CommonNav>
  <CommonHeader :user-name="employeeName"></CommonHeader>

  <MainItem w="calc(100% - 12rem)" h="calc(100vh - 10rem)" fld="row">
    <!-- 세션 리스트 Section -->
    <div class="session-list-container">
      <div class="session-list-header">
        <h2>대화 이력</h2>
        <button class="create-session-button" @click="createSession">
          <i class="fa-solid fa-comment-dots"></i> 대화 생성
        </button>
      </div>
      <ul class="session-list">
        <li
          v-for="session in sessions"
          :key="session.session_id"
          :class="{ active: selectedSession === session.session_id }"
          @click="selectSession(session.session_id)"
        >
          <span style="font-size: 1.4rem; font-weight: 500"
            ><strong>질문:</strong>
            {{ truncateText(session.first_question, 10) }}
          </span>
          <p><strong>생성일:</strong> {{ formatDate(session.created_at) }}</p>
        </li>
      </ul>
    </div>

    <!-- 챗봇 Section -->
    <div class="chatbot-section">
      <!-- 챗봇 히스토리 -->
      <div class="chatbot-history" :class="{ init: isInit }">
        <p v-if="isInit" class="help-text">무엇을 도와드릴까요?</p>
        <ul v-else class="chat-list">
          <li
            v-for="(item, index) in chatList"
            :key="index"
            class="chat-item"
            :class="{
              'user-chat': item.type === 'user',
              'bot-chat': item.type === 'bot',
            }"
          >
            <div v-if="item.type === 'bot'" class="item-header">
              <RobotIcon class="robot-icon"></RobotIcon>
              <CheckButton
                v-if="isCopied[index]"
                h="1.6rem"
                w="1.6rem"
                bgc="transparent"
                c="#888"
                fs="1.6rem"
              ></CheckButton>
              <CopyButton
                v-else
                h="1.6rem"
                w="1.6rem"
                bgc="transparent"
                c="#888"
                fs="1.6rem"
                @click="handleCopy(index, item.message)"
              ></CopyButton>
            </div>
            <span
              v-if="item.type === 'bot'"
              v-html="parseMarkdown(item.message)"
            ></span>
            <span v-else v-html="item.message"></span>
            <!-- 키워드에 따른 라우팅 버튼 -->
            <div
              v-if="
                item.type === 'bot' &&
                chatList[index].selectedKeyword &&
                chatList[index].selectedKeyword !== 'Nothing'
              "
              class="keyword-navigation"
            >
              <button @click="navigateToKeyword(index)" class="keyword-button">
                {{ keywordDict[chatList[index].selectedKeyword] }} 탭으로
                이동하기
              </button>
            </div>
          </li>
        </ul>
      </div>
    </div>

    <!-- 입력창 Section -->
    <div class="chat-container">
      <textarea
        class="chat-textarea"
        placeholder="메세지를 입력하세요."
        v-model="message"
        @input="adjustHeight"
        @keydown="handleKeydown"
        ref="textarea"
      ></textarea>
      <div class="btn-container">
        <!-- 로딩 상태 -->
        <template v-if="isLoading">
          <div class="spinner-wrapper">
            <i class="fas fa-spinner fa-spin"></i>
            <!-- 로딩 아이콘 -->
          </div>
        </template>
        <!-- 기본 상태 -->
        <template v-else>
          <ArrowUpButton
            class="submit-btn"
            h="3rem"
            w="3rem"
            bgc="#003566"
            br="50%"
            fs="1.6rem"
            @click="handleSubmit"
          />
        </template>
      </div>
    </div>
  </MainItem>
</template>

<script setup>
import CommonNav from '@/components/common/CommonNav.vue';
import CommonHeader from '@/components/common/CommonHeader.vue';
import MainItem from '@/components/semantic/MainItem.vue';
import ArrowUpButton from '@/components/buttons/ArrowUpButton.vue';
import CopyButton from '@/components/buttons/CopyButton.vue';
import CheckButton from '@/components/buttons/CheckButton.vue';
import RobotIcon from '@/components/icons/RobotIcon.vue';
import { ref, onMounted, nextTick } from 'vue';

import { marked } from 'marked';
import Prism from 'prismjs';
import 'prismjs/themes/prism.css';
import 'prismjs/components/prism-javascript';
import 'prismjs/components/prism-python';
import 'prismjs/components/prism-java';
import 'prismjs/components/prism-c';
import 'prismjs/components/prism-cpp';

import {
  chatbotQuery,
  getChatbotSessions,
  getSessionHistory,
} from '@/api/chatbot'; // 추가된 import

import { useRouter } from 'vue-router';
import { watch } from 'vue';

const router = useRouter();

const eid = ref(null);
const sessionId = ref('');
const selectedSessionId = ref('');
const employeeName = ref('');
const message = ref(''); // textarea 내용
const textarea = ref(null); // textarea DOM 참조
const isInit = ref(true);
const isCopied = ref([]);

const chatList = ref([]); // 현재 입력한 채팅 리스트

// 상태 변수
const sessions = ref([]); // 세션 목록
const chatHistory = ref([]); // 대화 이력
const selectedSession = ref(null); // 선택된 세션 ID
const userMessage = ref(''); // 사용자 입력 메시지
const isLoading = ref(false); // 로딩 상태 변수

const fetchAnswer = async (eid, sessionId, query) => {
  const formData = {
    query: query,
    employee_id: eid,
    session_id: sessionId,
  };

  try {
    isLoading.value = true; // 로딩 시작
    chatList.value.push({
      type: 'bot',
      message: 'AI가 응답을 생성중입니다. 잠시만 기다려 주세요...', // 로딩 중 메시지 추가
      selectedKeyword: null,
    });

    const response = await chatbotQuery(formData);

    console.log('챗봇 응답:', response);

    if (response && response.content) {
      // 로딩 중 메시지 제거
      chatList.value.pop();

      const sanitizedMessage = response.content.answer.replace(/\n+/g, '\n');
      const selectedKeyword = response.content.selected_keyword;
      chatList.value.push({
        type: 'bot',
        message: sanitizedMessage,
        selectedKeyword: selectedKeyword,
      });

      isCopied.value.push(false);

      nextTick(() => {
        scrollToBottom();
      });
    } else {
      chatList.value.pop(); // 로딩 중 메시지 제거
      chatList.value.push({
        type: 'bot',
        message: '챗봇에서 유효한 응답을 받지 못했습니다. 다시 시도해주세요.',
      });
    }
  } catch (error) {
    console.error('챗봇 응답 처리 중 오류:', error);
    chatList.value.pop(); // 로딩 중 메시지 제거
    chatList.value.push({
      type: 'bot',
      message: '오류가 발생했습니다. 잠시 후 다시 시도해주세요.',
      selectedKeyword: null,
    });
  } finally {
    isLoading.value = false; // 로딩 종료
  }
};

// 세션 선택 함수
const selectSession = async (sessionId) => {
  selectedSessionId.value = sessionId;
  selectedSession.value = sessionId; // 선택된 세션 ID 업데이트
  await fetchSessionHistory(sessionId); // 해당 세션의 대화 이력 가져오기
  isInit.value = false; // 초기 상태를 false로 변경 (대화 표시)
};

// 챗봇 세션 목록 가져오기
const fetchChatbotSessions = async () => {
  try {
    const response = await getChatbotSessions(eid.value); // API 호출
    if (response.success) {
      // 세션 목록을 created_at 기준으로 최신순 정렬
      sessions.value = response.content.sort(
        (a, b) => new Date(b.created_at) - new Date(a.created_at)
      );
    } else {
      console.warn('세션 목록이 비어있습니다.');
    }
  } catch (error) {
    console.error('챗봇 세션 목록 조회 실패:', error);
  }
};

//텍스트 청킹하기
const truncateText = (text, maxLength) => {
  if (text.length > maxLength) {
    return text.substring(0, maxLength) + '...';
  }
  return text;
};

// 날짜 포맷 함수
const formatDate = (dateString) => {
  const options = { year: 'numeric', month: 'short', day: 'numeric' };
  const date = new Date(dateString);
  return date.toLocaleDateString('ko-KR', options); // 한국어 형식으로 날짜 반환
};

// 특정 세션의 대화 이력 가져오기
const fetchSessionHistory = async (sessionId) => {
  try {
    const response = await getSessionHistory(sessionId); // API 호출

    if (response.success && response.content && response.content.length > 0) {
      chatList.value = response.content.map((item) => ({
        type: item.chatbot_type === 'HUMAN' ? 'user' : 'bot',
        message: item.chatbot_content,
        selectedKeyword: item.selected_keyword, // selected_keyword 추가
      }));

      isCopied.value = new Array(chatList.value.length).fill(false);
    } else {
      console.warn('대화 이력이 없습니다.');
      chatList.value = []; // 새 세션일 경우 빈 배열로 초기화
    }
  } catch (error) {
    console.error('대화 이력 조회 실패:', error);
    chatList.value = [
      {
        type: 'bot',
        message:
          '대화 이력을 불러오는 중 오류가 발생했습니다. 다시 시도해주세요.',
        selectedKeyword: null,
      },
    ];
  }
};

// 키워드와 경로 매핑
const keywordRoutes = {
  commute: { link: '/hr-basic/attendance/commute', subIdx: 0 },
  remote: { link: '/hr-basic/attendance/remote', subIdx: 1 },
  overtime: { link: '/hr-basic/attendance/overtime', subIdx: 2 },
  vacation: { link: '/hr-basic/attendance/vacation', subIdx: 3 },
  leave: { link: '/hr-basic/attendance/leave', subIdx: 4 },
  return: { link: '/hr-basic/attendance/return', subIdx: 5 },
  business: { link: '/hr-basic/attendance/business-trip', subIdx: 6 },
  dispatch: { link: '/hr-basic/attendance/dispatch', subIdx: 7 },
  salary: { link: '/hr-basic/salary', subIdx: 0 },
  contract: { link: '/hr-basic/document/contract', subIdx: 0 },
  certificate: { link: '/hr-basic/document/certificate', subIdx: 1 },
  department: { link: '/hr-basic/my-department/info/careers', subIdx: 0 },
  evaluation: { link: '/evaluation/personal', subIdx: 0 },
};

// 영어 키워드 매핑 (반대로 변경)
const keywordDict = {
  commute: '근퇴',
  remote: '재택',
  overtime: '초과근무',
  vacation: '휴가',
  leave: '휴직',
  return: '복직',
  business: '출장',
  dispatch: '파견',
  salary: '급여',
  contract: '계약',
  certificate: '증명',
  department: '부서',
  evaluation: '평가',
};

// 라우팅 함수
const navigateToKeyword = (index) => {
  const selectedKeyword = chatList.value[index].selectedKeyword;
  if (selectedKeyword !== null && selectedKeyword !== 'Nothing') {
    const route = keywordRoutes[selectedKeyword].link;
    localStorage.setItem('subIdx', keywordRoutes[selectedKeyword].subIdx);
    if (route) {
      router.push(route);
    } else {
      console.error('적절한 경로가 없습니다.');
    }
  } else {
    console.error('선택된 키워드가 매핑되지 않았습니다.');
  }
};

const adjustHeight = () => {
  const el = textarea.value; // textarea 요소

  if (!el) return;

  // 텍스트가 없으면 높이를 초기화
  const initHeight = 80;
  el.style.height = initHeight + 'px';

  // 줄 높이를 구하고, 최대 3줄까지만 늘어남
  const lineHeight = parseFloat(window.getComputedStyle(el).lineHeight); // 줄 높이 계산
  const maxLines = 3; // 최대 3줄
  const newHeight = Math.min(
    el.scrollHeight,
    initHeight + lineHeight * maxLines
  ); // 최대 4줄까지만 높이 설정

  // 높이 조정
  el.style.height = `${newHeight}px`; // 새로 계산한 높이 적용

  // 5줄 이상부터는 스크롤을 활성화
  if (el.scrollHeight > newHeight) {
    el.style.overflowY = 'auto'; // 스크롤 활성화
  } else {
    el.style.overflowY = 'hidden'; // 스크롤 비활성화
  }
};

const generateSessionId = () => {
  const array = new Uint8Array(24); // 24바이트 랜덤값
  crypto.getRandomValues(array); // 랜덤 값을 생성
  return Array.from(array, (byte) => byte.toString(16).padStart(2, '0')).join(
    ''
  ); // 16진수 문자열로 변환
};

const createSession = () => {
  // 새로고침
  window.location.reload(); // 브라우저 새로고침
};

// watch를 사용하여 isInit 상태 변화 감지
watch(isInit, async (newValue) => {
  if (!newValue) {
    // isInit이 false로 변경되었을 때 실행
    await fetchChatbotSessions(); // 세션 목록 다시 불러오기
  }
});

const handleKeydown = (event) => {
  if (isLoading.value) {
    event.preventDefault(); // 로딩 중일 경우 기본 동작 차단
    return;
  }

  if (event.key === 'Enter') {
    if (event.shiftKey) {
      // shift + enter일 경우 줄바꿈
      return;
    } else {
      // enter일 경우 submit
      event.preventDefault(); // 기본 enter 동작 방지 (줄바꿈 방지)
      handleSubmit(); // 메세지 제출
    }
  }
};

const handleCopy = (index, message) => {
  navigator.clipboard
    .writeText(message)
    .then(() => {
      isCopied.value[index] = true;
      setTimeout(() => {
        handleReturn(index); // 500ms 후 다시 초기 상태로
      }, 3000);
    })
    .catch((error) => {
      console.error('클립보드 복사 실패:', error);
    });
};

const handleReturn = (index) => {
  isCopied.value[index] = false;
};

const handleSubmit = () => {
  if (!message.value) return;
  isInit.value = false;
  chatList.value.push({
    type: 'user',
    message: message.value,
    selectedKeyword: null,
  });
  fetchAnswer(eid.value, selectedSessionId.value, message.value);
  message.value = '';

  // 응답 후 스크롤을 최하단으로 이동
  nextTick(() => {
    adjustHeight(); // 메시지 제출 후 textarea 크기 조정
    scrollToBottom();
  });
};

// 스크롤을 강제로 끝까지 이동
const scrollToBottom = () => {
  const sessionList = document.querySelector('.chat-list');
  if (sessionList) {
    sessionList.scrollTo({
      top: sessionList.scrollHeight,
      behavior: 'smooth', // 부드럽게 스크롤 이동
    });
  }
};

const decodeHtmlEntities = (text) => {
  const textarea = document.createElement('textarea');
  textarea.innerHTML = text;
  return textarea.value;
};

// 마크다운 파싱 함수
const parseMarkdown = (text) => {
  const html = marked(text); // 마크다운을 HTML로 변환

  // 하이라이팅 처리
  const highlightedHtml = html.replace(
    /<pre><code class="language-(\w+)">([\s\S]*?)<\/code><\/pre>/g,
    (match, language, code) => {
      // HTML 엔티티를 디코딩
      const decodedCode = decodeHtmlEntities(code);

      // 디코딩된 코드에 하이라이팅 적용
      const highlightedCode = Prism.highlight(
        decodedCode,
        Prism.languages[language],
        language
      );
      return `<pre><code class="language-${language}">${highlightedCode}</code></pre>`;
    }
  );

  // 렌더링된 HTML을 삽입 후 하이라이팅
  nextTick(() => {
    const codeBlocks = document.querySelectorAll('pre code');
    codeBlocks.forEach((block) => {
      Prism.highlightElement(block); // 각 코드 블록에 하이라이팅 적용
    });
  });

  return highlightedHtml;
};

onMounted(() => {
  eid.value = localStorage.getItem('employeeId');
  if (!eid.value) {
    alert('로그인이 필요합니다.');
    router.push('/login');
  }
  employeeName.value = localStorage.getItem('employeeName');
  sessionId.value = generateSessionId();
  selectedSessionId.value = sessionId.value;

  // 챗봇 세션 목록 가져오기
  fetchChatbotSessions();

  // 특정 세션의 대화 이력 가져오기 (예: 초기 로딩 시 특정 세션 ID로 가져오기)
  fetchSessionHistory(selectedSessionId.value);
});
</script>

<style scoped>
/* 세션 리스트 스타일 */
.session-list-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow-y: auto;
  border-right: 1px solid #ddd;
  width: 18%; /* 기존 세션 리스트의 너비 */
}

/* 헤더 영역 스타일 */
.session-list-header {
  display: flex;
  justify-content: space-between; /* 제목과 버튼을 양쪽에 배치 */
  align-items: center;
  padding: 1rem; /* 내부 여백 */
}

.session-list-header h2 {
  font-size: 2rem; /* 제목 크기 */
  font-weight: bold;
  margin: 0; /* 여백 제거 */
  align-items: center;
  height: 100%; /* 부모 컨테이너 높이 기준으로 정렬 */
}

.create-session-button {
  background-color: #003566; /* 버튼 배경색 */
  color: #fff; /* 버튼 텍스트 색상 */
  border: none;
  border-radius: 0.5rem;
  padding: 0.4rem 0.8rem;
  font-size: 1.2rem; /* 버튼 텍스트 크기 */
  margin-top: 0.6rem; /* 상단 간격 추가 */
  cursor: pointer;
  transition: background-color 0.3s ease; /* 배경색 전환 효과 */
}

.create-session-button:hover {
  background-color: #0056b3; /* 호버 시 배경색 */
}

.session-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.session-list li {
  padding: 1.5rem 1rem;
  margin-top: 1rem;
  margin-bottom: 1rem;
  margin-right: 1rem;
  background-color: #f7f7f7f7;
  border-radius: 4px;
  cursor: pointer;
  transition:
    background-color 0.3s ease,
    box-shadow 0.3s ease;
}

.session-list li:hover {
  background-color: #e0e0e0;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.1);
}

.session-list li.active {
  background-color: #787878;
  color: #f7f7f7f7;
}

.session-list li p,
.session-list li span {
  margin: 0.5rem 0;
  font-size: 1.4rem;
  line-height: 1.6;
  word-wrap: break-word; /* 긴 텍스트 줄바꿈 */
}
/* 챗봇 영역 스타일 */
.chatbot-section {
  position: static; /* 상위 컨테이너의 기준 제거 */
  flex: 1; /* 남은 공간을 모두 차지 */
  display: flex;
  flex-direction: column; /* 세로 정렬 */
  height: calc(100vh - 14rem); /* 입력창 높이를 제외한 나머지 높이 */
  background-color: #ffffff;
  overflow: hidden; /* 넘치는 내용 숨김 */
}

.init {
  align-items: center;
  justify-content: center;
}

.help-text {
  text-align: center;
  font-size: 3.2rem;
  font-weight: 500;
  margin-bottom: 4rem;
}

/* 입력창 고정 */
.chat-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  position: fixed; /* 화면 하단에 고정 */
  bottom: 2%; /* 화면 하단에서 0만큼 띄움 */
  left: 32%; /* 히스토리 섹션의 너비만큼 띄움 */
  width: calc(100% - 40%); /* 히스토리 제외 나머지 공간 차지 */
  background-color: #f7f7f7;
  border-radius: 1rem;
  padding: 1rem;
  z-index: 10; /* 다른 요소 위로 */
}

/* 챗봇 히스토리 */
.chatbot-history {
  display: flex;
  flex-direction: column; /* 세로 방향으로 배치 */
  height: calc(100% - 10rem); /* 부모 요소에 맞춰 높이 설정 */
  padding: 1rem;
  background-color: #ffffff;
}

/* 채팅 목록 */
.chat-list {
  display: flex;
  flex-direction: column;
  height: 100%;
  width: 100%;
  gap: 3rem;
  padding: 1rem;
  overflow-y: auto;
}

/* 대화 아이템 스타일 */
.chat-item {
  display: flex;
  flex-direction: column;
  margin-bottom: 1.8rem; /* 메시지 간 간격 */
  border-radius: 1.5rem;
  max-width: 80rem;
  padding: 1rem;
  font-size: 1.6rem;
}

.bot-chat {
  margin-left: 10%;
  align-self: flex-start; /* 왼쪽 정렬 */
  width: auto; /* 너비를 컨텐츠에 맞게 */
  max-width: 48%; /* 최대 너비 설정 */
  background-color: #f7f7f7; /* 봇 메시지 배경색 */
  border-radius: 7px; /* 둥근 모서리 */
  padding: 1rem 1.5rem; /* 내부 여백 */
  box-shadow: 0 0.4rem 0.8rem rgba(0, 0, 0, 0.1);
}

.user-chat {
  margin-right: 10%;
  align-self: flex-end; /* 오른쪽 정렬 */
  width: auto; /* 너비를 컨텐츠에 맞게 */
  max-width: 48%; /* 최대 너비 설정 */
  background-color: #e6e6e6; /* 사용자 메시지 배경색 */
  border-radius: 7px; /* 둥근 모서리 */
  padding: 1rem 1.5rem; /* 내부 여백 */
  box-shadow: 0 0.4rem 0.8rem rgba(0, 0, 0, 0.1);
}

.chat-item span {
  display: block;
  word-wrap: break-word; /* 구버전 호환 */
  overflow-wrap: break-word; /* 긴 단어도 줄바꿈 */
  white-space: pre-wrap; /* 공백 및 줄바꿈 처리 */
}

.chat-textarea {
  width: calc(100% - 5rem);
  height: 80px;
  border-radius: 0.5rem;
  padding: 0.8rem;
  font-size: 1.4rem;
  resize: none; /* 크기 조정 비활성화 */
  line-height: 16px;
}

/* 키워드 네비게이션 버튼 */
.keyword-navigation {
  margin-top: 1rem;
  text-align: left;
}

.keyword-button {
  display: inline-block; /* 버튼을 인라인으로 */
  margin-left: 0; /* 왼쪽 여백을 없앰 */
  padding: 0.6rem 1.2rem;
  border: none;
  border-radius: 3px;
  background-color: #003356;
  color: #fff;
  font-size: 1.2rem;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.keyword-button:hover {
  background-color: #0056b3;
}

.keyword-button:active {
  background-color: #003d80;
}
/* 키워드 네비게이션 버튼 끝 */

/* 제출 버튼 스타일 */
.submit-btn {
  width: 3rem;
  height: 3rem;
  border-radius: 50%;
  background-color: #003566;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.item-header {
  display: flex; /* Flexbox 활성화 */
  flex-direction: row; /* 기본값이지만 명시적으로 설정 */
  justify-content: space-between;
  align-items: center;
  padding-bottom: 1rem;
}

.item-header i {
  font-size: 1.6rem;
}

/* 스크롤바 스타일 */
.chat-list,
.session-list-section {
  scrollbar-width: thin;
  scrollbar-color: #ccc #f0f0f0;
}

.chat-list::-webkit-scrollbar,
.session-list-section::-webkit-scrollbar {
  width: 8px;
}

.chat-list::-webkit-scrollbar-thumb,
.session-list-section::-webkit-scrollbar-thumb {
  background-color: #ccc;
  border-radius: 4px;
}

.spinner-wrapper {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 3rem;
  width: 3rem;
  background-color: #003566; /* 로딩 배경색 */
  border-radius: 50%;
  font-size: 1.6rem;
}

.fa-spinner {
  animation: spin 1s linear infinite; /* 기본 Font Awesome 스타일 */
  font-size: 1.5rem;
  color: #ffffff; /* 아이콘 색상 */
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
