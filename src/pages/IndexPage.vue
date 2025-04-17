<template>
  <q-page class="q-mt-lg q-pr-sm q-pl-sm">
    <!-- 헤더 -->
    <div class="q-mb-md">
      <div class="row items-center">
        <h2 class="text-h5 text-primary text-weight-bold q-ma-none">디어발레 보강 예약</h2>
        <!-- 관리자 모드 버튼 (숨겨진 버튼 - 모바일 친화적) -->
        <div class="admin-button-wrapper q-ml-xs" @click.stop="longPressCount++">
          <q-icon 
            name="pets" 
            size="16px"
            color="grey"
            class="admin-icon"
          />
        </div>
      </div>
      <div class="row justify-end q-mt-md">
        <!-- 예약 확인 버튼 -->
        <q-btn 
          flat 
          dense 
          color="secondary" 
          icon="event_note"
          label="예약 확인"
          @click="showCheckDialog = true" 
        />
      </div>
    </div>

    <!-- 메인 캘린더 -->
    <q-card bordered>
      <q-card-section class="q-pa-none">
        <q-date
        flat
          v-model="selectedDate"
          :events="eventDates"
          event-color="secondary"
          color="secondary"
          :locale="locale"
          minimal
          class="m__calendar"
          @update:model-value="onDateSelected"
        />
      </q-card-section>
      <q-card-section class="text-center text-secondary">
        <div>날짜를 선택하면 클래스를 확인해 주세요.</div>
        <div class="text-caption text-grey">{{ eventDates.length > 0 ? '표시된 날짜에 예약 가능한 클래스가 있습니다' : '' }}</div>
      </q-card-section>
    </q-card>

    <!-- 관리자 로그인 다이얼로그 -->
    <q-dialog v-model="showAdminLoginDialog" position="bottom">
      <q-card style="width: 100%; max-width: 400px">
        <q-card-section class="row items-center">
          <div class="text-h6">관리자 인증</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>

        <q-card-section class="q-pt-none">
          <q-input 
            v-model="adminPassword" 
            type="password" 
            label="관리자 비밀번호" 
            outlined 
            dense
            @keyup.enter="loginAdmin"
          />
        </q-card-section>

        <q-card-actions align="right">
          <q-btn flat label="취소" color="negative" v-close-popup />
          <q-btn flat label="확인" color="positive" @click="loginAdmin" />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <!-- 선택한 날짜의 클래스 목록 다이얼로그 -->
    <q-dialog v-model="showClassListDialog" position="bottom">
      <q-card style="width: 100%; max-width: 500px; height: 300px;">
        <q-card-section class="row items-center">
          <div class="text-h6">{{ formatSelectedDate }}</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>

        <!-- 관리자 모드일 때 클래스 등록 폼 -->
        <div v-if="isAdminMode">
          <q-separator />
          <q-card-section>
            <div class="text-subtitle2">클래스 등록</div>

            <div class="row q-col-gutter-md q-mt-sm">
              <div class="col-12 col-sm-6">
                <q-select
                  v-model="scheduleForm.class_id"
                  :options="classes"
                  option-value="id"
                  option-label="name"
                  outlined
                  dense
                  label="클래스 선택"
                  emit-value
                  map-options
                  :rules="[val => !!val || '클래스를 선택해주세요']"
                />
              </div>
              <div class="col-12 col-sm-3">
                <q-input
                  v-model="scheduleForm.start_time"
                  outlined
                  dense
                  type="time"
                  label="시작 시간"
                  :rules="[val => !!val || '시작 시간을 입력해주세요']"
                />
              </div>
              <div class="col-12 col-sm-3">
                <q-input
                  v-model="scheduleForm.end_time"
                  outlined
                  dense
                  type="time"
                  label="종료 시간"
                  :rules="[val => !!val || '종료 시간을 입력해주세요']"
                />
              </div>
            </div>

            <div class="row q-mt-md">
              <div class="col-12 text-right">
                <q-btn 
                  color="primary" 
                  label="클래스 등록" 
                  @click="createSchedule" 
                />
              </div>
            </div>
          </q-card-section>

          <!-- 등록된 클래스가 없을 때 -->
          <q-banner v-if="!classes.length" class="bg-amber-1">
            <template v-slot:avatar>
              <q-icon name="warning" color="warning" />
            </template>
            등록된 클래스가 없습니다. 먼저 클래스를 생성해주세요.
            <template v-slot:action>
              <q-btn flat color="primary" label="클래스 생성" @click="showClassFormDialog = true" />
            </template>
          </q-banner>
        </div>

        <!-- 클래스 목록 -->
        <!-- <q-separator />
        <q-card-section>
          <div class="text-subtitle2">{{ isAdminMode ? '등록된 클래스' : '' }}</div>
        </q-card-section> -->

        <q-list separator>
          <q-item
            v-for="schedule in schedulesForDate"
            :key="schedule.id"
            :class="{ 'bg-grey-2': schedule.available_slots <= 0 }"
            :style="schedule.color ? { borderLeft: `4px solid ${schedule.color}` } : ''"
          >
            <q-item-section>
              <q-item-label class="text-weight-bold q-pb-md">{{ schedule.class_name }}</q-item-label>
              <q-item-label>
                시간: {{ schedule.start_time }} - {{ schedule.end_time }}
              </q-item-label>
              <q-item-label>
                남은 자리: {{ schedule.available_slots }} / {{ getClassMaxCapacity(schedule) }}
              </q-item-label>
            </q-item-section>

            <q-item-section side>
              <!-- 관리자 모드일 때 삭제 버튼 -->
              <q-btn
                v-if="isAdminMode"
                color="negative"
                flat
                dense
                icon="delete"
                @click="deleteSchedule(schedule.id)"
              />
              
              <!-- 일반 모드일 때 예약 버튼 -->
              <q-btn
                v-else
                color="primary"
                outline
                :disable="schedule.available_slots <= 0"
                :label="schedule.available_slots <= 0 ? '마감' : '예약'"
                @click="showBookingForm(schedule)"
              />
            </q-item-section>
          </q-item>
          <q-item v-if="!schedulesForDate.length">
            <q-item-section>
              <q-item-label class="text-grey text-center q-py-md">
                {{ isAdminMode ? '등록된 클래스가 없습니다.' : '예약 가능한 클래스가 없습니다.' }}
              </q-item-label>
            </q-item-section>
          </q-item>
        </q-list>

        <!-- 예약 관리 (관리자 모드) -->
        <div v-if="isAdminMode && bookingsForDate.length > 0">
          <q-separator />
          <q-card-section>
            <div class="text-subtitle2">예약 관리</div>
          </q-card-section>

          <q-list separator>
            <q-item
              v-for="booking in bookingsForDate"
              :key="booking.id"
              :class="{
                'bg-green-1': booking.status === 'confirmed',
                'bg-red-1': booking.status === 'cancelled',
                'bg-amber-1': booking.status === 'pending'
              }"
            >
              <q-item-section>
                <q-item-label class="text-weight-bold">{{ booking.name }}</q-item-label>
                <q-item-label caption>
                  {{ booking.class_name }} | {{ booking.start_time }} - {{ booking.end_time }}
                </q-item-label>
                <q-item-label caption>
                  <span>상태: {{ getStatusText(booking.status) }}</span>
                  <span v-if="booking.phone" class="q-ml-sm">연락처: {{ booking.phone }}</span>
                </q-item-label>
              </q-item-section>

              <q-item-section side v-if="booking.status === 'pending'">
                <div class="row q-gutter-xs">
                  <q-btn
                    color="negative"
                    flat
                    dense
                    icon="close"
                    @click="updateBookingStatus(booking.id, 'cancelled')"
                  />
                  <q-btn
                    color="positive"
                    flat
                    dense
                    icon="check"
                    @click="updateBookingStatus(booking.id, 'confirmed')"
                  />
                </div>
              </q-item-section>
            </q-item>
          </q-list>
        </div>
      </q-card>
    </q-dialog>

    <!-- 클래스 생성 다이얼로그 -->
    <q-dialog v-model="showClassFormDialog" position="bottom">
      <q-card style="width: 100%; max-width: 400px">
        <q-card-section class="row items-center">
          <div class="text-h6">새 클래스 생성</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>

        <q-card-section>
          <q-input
            v-model="classForm.name"
            outlined
            dense
            label="클래스 이름"
            :rules="[val => !!val || '클래스 이름을 입력해주세요']"
          />
          <q-input
            v-model.number="classForm.max_capacity"
            type="number"
            outlined
            dense
            label="최대 인원"
            class="q-mt-sm"
            :rules="[val => val > 0 || '1명 이상 입력해주세요']"
          />
          <div class="row items-center q-mt-sm">
            <div class="col-4">
              <div class="text-subtitle2">색상</div>
            </div>
            <div class="col-8">
              <q-color v-model="classForm.color" />
            </div>
          </div>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn flat label="취소" color="negative" v-close-popup />
          <q-btn flat label="생성" color="positive" @click="createClass" />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <!-- 예약 신청 다이얼로그 -->
    <q-dialog v-model="showBookingDialog" position="bottom">
      <q-card style="width: 100%; max-width: 400px">
        <q-card-section class="row items-center">
          <div class="text-h6">클래스 예약</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>

        <q-card-section v-if="selectedSchedule">
          <div class="text-subtitle2">예약 정보</div>
          <div class="q-mt-sm">
            <div><span class="text-weight-bold">날짜:</span> {{ formatSelectedDate }}</div>
            <div><span class="text-weight-bold">클래스:</span> {{ selectedSchedule.class_name }}</div>
            <div><span class="text-weight-bold">시간:</span> {{ selectedSchedule.start_time }} - {{ selectedSchedule.end_time }}</div>
            <div><span class="text-weight-bold">남은 자리:</span> {{ selectedSchedule.available_slots }}</div>
          </div>
        </q-card-section>

        <q-form @submit="submitBooking">
          <q-card-section>
            <div class="text-subtitle2">예약자 정보</div>
            <q-input
              v-model="bookingForm.name"
              outlined
              dense
              label="이름"
              class="q-mt-sm"
              :rules="[val => !!val || '이름을 입력해주세요']"
            />
            
            <q-input
              v-model="bookingForm.password"
              outlined
              dense
              label="비밀번호 (4자리 숫자)"
              type="password"
              mask="####"
              class="q-mt-sm"
              :rules="[
                val => !!val || '비밀번호를 입력해주세요',
                val => val.length === 4 || '4자리 숫자를 입력해주세요'
              ]"
              hint="예약 확인 및 취소 시 필요합니다"
            />
            
            <q-input
              v-model="bookingForm.phone"
              outlined
              dense
              label="연락처 (선택)"
              class="q-mt-sm"
              hint="예: 010-1234-5678"
            />
          </q-card-section>

          <q-card-actions align="right">
            <q-btn flat label="취소" color="negative" v-close-popup />
            <q-btn type="submit" flat label="예약 신청" color="positive" />
          </q-card-actions>
        </q-form>
      </q-card>
    </q-dialog>

    <!-- 예약 확인 다이얼로그 -->
    <q-dialog v-model="showCheckDialog" position="bottom">
      <q-card style="width: 100%; max-width: 400px">
        <q-card-section class="row items-center">
          <div class="text-h6">예약 확인</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>

        <q-form @submit="checkBooking">
          <q-card-section>
            <q-input
              v-model="checkForm.name"
              outlined
              dense
              label="이름"
              :rules="[val => !!val || '이름을 입력해주세요']"
            />
            
            <q-input
              v-model="checkForm.password"
              outlined
              dense
              label="비밀번호"
              type="password"
              class="q-mt-sm"
              :rules="[val => !!val || '비밀번호를 입력해주세요']"
            />
          </q-card-section>

          <q-card-actions align="right">
            <q-btn flat label="취소" color="negative" v-close-popup />
            <q-btn type="submit" flat label="예약 확인" color="positive" />
          </q-card-actions>
        </q-form>
      </q-card>
    </q-dialog>

    <!-- 예약 목록 다이얼로그 -->
    <q-dialog v-model="showMyBookingsDialog" position="bottom">
      <q-card style="width: 100%; max-width: 400px" class="q-pb-none">
        <q-card-section class="row items-center">
          <div class="text-h6">등록된 클래스</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>

        <q-list separator>
          <q-item
            v-for="booking in myBookings"
            :key="booking.id"
            :class="{
              'bg-info': booking.status === 'confirmed',
              'bg-info': booking.status === 'cancelled',
              'bg-info': booking.status === 'pending'
            }"
          >
            <q-item-section>
              <q-item-label class="text-weight-bold">{{ booking.class_name }}</q-item-label>
              <q-item-label>
                {{ formatDate(booking.date) }} | {{ booking.start_time }} - {{ booking.end_time }}
              </q-item-label>
              <q-item-label>
                상태: {{ getStatusText(booking.status) }}
              </q-item-label>
            </q-item-section>

            <q-item-section side v-if="booking.status !== 'cancelled'">
              <q-btn
                color="negative"
                outline
                dense
                label="취소"
                @click="cancelMyBooking(booking.id)"
              />
            </q-item-section>
          </q-item>
          <q-item v-if="!myBookings.length">
            <q-item-section>
              <q-item-label class="text-grey text-center q-py-md">
                예약 정보가 없습니다.
              </q-item-label>
            </q-item-section>
          </q-item>
        </q-list>
      </q-card>
    </q-dialog>
  </q-page>
</template>
<script setup>
import { ref, computed, onMounted, watchEffect, watch } from 'vue';
import { useQuasar } from 'quasar';
//import axios from 'axios';

const $q = useQuasar();
//const API_URL = 'http://localhost:5000/api';

// 로케일 설정
const locale = ref({
  days: ['일요일', '월요일', '화요일', '수요일', '목요일', '금요일', '토요일'],
  daysShort: ['일', '월', '화', '수', '목', '금', '토'],
  months: ['1월', '2월', '3월', '4월', '5월', '6월', '7월', '8월', '9월', '10월', '11월', '12월'],
  monthsShort: ['1월', '2월', '3월', '4월', '5월', '6월', '7월', '8월', '9월', '10월', '11월', '12월']
});

// 상태 관리
// 오늘 날짜로 초기화 (YYYY/MM/DD 형식)
const today = new Date();
const formattedToday = `${today.getFullYear()}/${String(today.getMonth() + 1).padStart(2, '0')}/${String(today.getDate()).padStart(2, '0')}`;
const selectedDate = ref();

const eventDates = ref([]);  // 예약이 있는 날짜 배열

// 다이얼로그 상태
const showClassListDialog = ref(false);  // 클래스 목록 다이얼로그
const showBookingDialog = ref(false);    // 예약 신청 다이얼로그
const showCheckDialog = ref(false);      // 예약 확인 다이얼로그
const showMyBookingsDialog = ref(false); // 예약 목록 다이얼로그
const showAdminLoginDialog = ref(false); // 관리자 로그인 다이얼로그
const showClassFormDialog = ref(false);  // 클래스 생성 다이얼로그

// 데이터
const classes = ref([]);
const schedulesForDate = ref([]);
const bookingsForDate = ref([]);
const myBookings = ref([]);

// 관리자 모드 진입을 위한 시크릿 카운터 (모바일 최적화)
const longPressCount = ref(0);
const longPressTimer = ref(null);

// 폼 데이터
const bookingForm = ref({
  name: '',
  password: '',
  phone: ''
});

const checkForm = ref({
  name: '',
  password: ''
});

const classForm = ref({
  name: '',
  max_capacity: 10,
  color: '#4CAF50'
});

const scheduleForm = ref({
  class_id: '',
  start_time: '',
  end_time: ''
});

// 관리자 모드 관련
const isAdminMode = ref(false);
const adminPassword = ref('');

// 선택한 스케줄
const selectedSchedule = ref(null);

// 계산된 속성
const formatSelectedDate = computed(() => {
  if (!selectedDate.value) return '';
  
  const dateParts = selectedDate.value.split('/');
  if (dateParts.length !== 3) return selectedDate.value;
  
  const year = dateParts[0];
  const month = dateParts[1];
  const day = dateParts[2];
  
  // 요일 계산
  const date = new Date(year, month - 1, day);
  const weekday = locale.value.daysShort[date.getDay()];
  
  return `${year}년 ${month}월 ${day}일 (${weekday})`;
});

// 관리자 버튼 롱프레스 감지 로직
watch(longPressCount, (newVal) => {
  if (newVal >= 5) {
    toggleAdminMode();
    longPressCount.value = 0;
  } else if (newVal > 0) {
    // 3초 내에 추가 터치가 없으면 카운트 리셋
    if (longPressTimer.value) clearTimeout(longPressTimer.value);
    longPressTimer.value = setTimeout(() => {
      longPressCount.value = 0;
    }, 3000);
  }
});

// 유틸리티 함수
const formatApiDateString = (dateStr) => {
  // Quasar QDate는 YYYY/MM/DD 형식, API는 YYYY-MM-DD 형식을 사용
  return dateStr.replace(/\//g, '-');
};

const formatDateForQDate = (dateStr) => {
  // API의 YYYY-MM-DD 형식을 Quasar QDate의 YYYY/MM/DD 형식으로 변환
  return dateStr.replace(/-/g, '/');
};

const formatDate = (dateStr) => {
  if (!dateStr) return '';
  
  const date = new Date(dateStr);
  const year = date.getFullYear();
  const month = date.getMonth() + 1;
  const day = date.getDate();
  const weekday = locale.value.daysShort[date.getDay()];
  
  return `${year}년 ${month}월 ${day}일 (${weekday})`;
};

const getClassMaxCapacity = (schedule) => {
  // classes 배열에서 해당 class_id를 가진 클래스를 찾아 max_capacity 반환
  const classInfo = classes.value.find(c => c.id === schedule.class_id);
  return classInfo ? classInfo.max_capacity : schedule.max_capacity || 0;
};

const getStatusText = (status) => {
  const statusMap = {
    'pending': '승인 대기',
    'confirmed': '예약 확정',
    'cancelled': '예약 취소'
  };
  return statusMap[status] || status;
};

// 데이터 로드 함수
const fetchClasses = async () => {
  try {
    // const response = await axios.get(`${API_URL}/classes`);
    // classes.value = response.data;
    
    // // 클래스가 있으면 scheduleForm의 기본값 설정
    // if (classes.value.length > 0) {
    //   scheduleForm.value.class_id = classes.value[0].id;
    // }
  } catch (error) {
    console.error('클래스 로드 실패:', error);
    $q.notify({
      color: 'negative',
      message: '클래스 정보를 불러오는데 실패했습니다.',
      icon: 'error'
    });
  }
};


const fetchEventDates = async () => {
  try {
    // 실제 구현에서는 API에서 예약이 있는 날짜 목록을 가져와야 함
    // 예시 데이터
    eventDates.value = [
      formatDateForQDate('2025-04-18'),
      formatDateForQDate('2025-04-20'),
      formattedToday // 오늘 날짜도 이벤트로 표시
    ];
  } catch (error) {
    console.error('이벤트 날짜 로드 실패:', error);
  }
};

const fetchSchedulesForDate = async (date) => {
  if (!date) return;
  
  try {
    // 실제 API 호출 코드
    // const response = await axios.get(`${API_URL}/schedules/date/${apiDateString}`);
    // schedulesForDate.value = response.data;

    // 예시 데이터 (실제 구현에서는 제거)
    if (date === formattedToday) {
      // 오늘 날짜에 맞는 예시 데이터
      schedulesForDate.value = [
        { id: 1, class_id: 1, class_name: '초급반', start_time: '10:00', end_time: '11:30', available_slots: 3, max_capacity: 5, color: '#E3A7C4' },
        { id: 2, class_id: 2, class_name: '중급반', start_time: '13:00', end_time: '14:30', available_slots: 2, max_capacity: 5, color: '#E3A7C4' }
      ];
    } else if (date === '2025/04/18') {
      schedulesForDate.value = [
        { id: 3, class_id: 1, class_name: '초급반', start_time: '10:00', end_time: '11:30', available_slots: 3, max_capacity: 5, color: '#E3A7C4' },
        { id: 4, class_id: 2, class_name: '중급반', start_time: '13:00', end_time: '14:30', available_slots: 0, max_capacity: 5, color: '#E3A7C4' }
      ];
    } else if (date === '2025/04/20') {
      schedulesForDate.value = [
        { id: 5, class_id: 3, class_name: '고급반', start_time: '15:00', end_time: '16:30', available_slots: 2, max_capacity: 4, color: '#E3A7C4' }
      ];
    } else {
      schedulesForDate.value = [];
    }
    
    // 데이터가 있으면 클래스 목록 다이얼로그 표시
    if (schedulesForDate.value.length > 0 || isAdminMode.value) {
      showClassListDialog.value = true;
    } else {
      $q.notify({
        color: 'info',
        message: '해당 날짜에 예약 가능한 클래스가 없습니다.',
        icon: 'info'
      });
    }
  } catch (error) {
    console.error('날짜별 스케줄 로드 실패:', error);
    schedulesForDate.value = [];
  }
};

const fetchBookingsForDate = async () => {
  if (!isAdminMode.value || !selectedDate.value) return;
  
  try {
    const apiDateString = formatApiDateString(selectedDate.value);
    // 실제 API 호출 코드
    // const response = await axios.post(`${API_URL}/admin/bookings`, {
    //   admin_password: localStorage.getItem('adminPassword'),
    //   date: apiDateString
    // });
    // bookingsForDate.value = response.data;
    
    // 예시 데이터 (실제 구현에서는 제거)
    if (selectedDate.value === '2025/04/18' || selectedDate.value === formattedToday) {
      bookingsForDate.value = [
        { 
          id: 1, 
          name: '김지수', 
          class_id: 1, 
          class_name: '초급반',
          date: apiDateString,
          start_time: '10:00', 
          end_time: '11:30',
          status: 'pending',
          phone: '010-1234-5678'
        },
        { 
          id: 2, 
          name: '이민호', 
          class_id: 2, 
          class_name: '중급반',
          date: apiDateString,
          start_time: '13:00', 
          end_time: '14:30',
          status: 'confirmed',
          phone: '010-9876-5432'
        }
      ];
    } else {
      bookingsForDate.value = [];
    }
  } catch (error) {
    console.error('예약 로드 실패:', error);
    bookingsForDate.value = [];
  }
};

// 이벤트 핸들러
const onDateSelected = (date) => {
  if (!date) return;
  
  fetchSchedulesForDate(date);
  
  if (isAdminMode.value) {
    fetchBookingsForDate();
  }
};

const showBookingForm = (schedule) => {
  selectedSchedule.value = schedule;
  bookingForm.value = {
    name: '',
    password: '',
    phone: ''
  };
  showClassListDialog.value = false; // 클래스 목록 다이얼로그 닫기
  showBookingDialog.value = true;    // 예약 다이얼로그 열기
};

const submitBooking = async () => {
  if (!selectedSchedule.value) return;
  
  // 비밀번호 검증
  if (!/^\d{4}$/.test(bookingForm.value.password)) {
    $q.notify({
      color: 'negative',
      message: '비밀번호는 4자리 숫자여야 합니다.',
      icon: 'error'
    });
    return;
  }
  
  try {
    // 실제 API 호출 코드
    // const response = await axios.post(`${API_URL}/bookings`, {
    //   name: bookingForm.value.name,
    //   password: bookingForm.value.password,
    //   phone: bookingForm.value.phone,
    //   schedule_id: selectedSchedule.value.id
    // });
    
    // 예시 응답 (실제 구현에서는 제거)
    const response = { data: { message: '예약이 신청되었습니다. 관리자 승인 후 확정됩니다.' } };
    
    $q.notify({
      color: 'positive',
      message: response.data.message,
      icon: 'check_circle'
    });
    
    showBookingDialog.value = false;
  } catch (error) {
    console.error('예약 실패:', error);
    $q.notify({
      color: 'negative',
      message: error.response?.data?.message || '예약 처리 중 오류가 발생했습니다.',
      icon: 'error'
    });
  }
};

const checkBooking = async () => {
  try {
    // 실제 API 호출 코드
    // const response = await axios.post(`${API_URL}/bookings/check`, {
    //   name: checkForm.value.name,
    //   password: checkForm.value.password
    // });
    // myBookings.value = response.data;
    
    // 예시 데이터 (실제 구현에서는 제거)
    myBookings.value = [
      { 
        id: 1, 
        class_name: '초급반',
        date: '2025-04-18',
        start_time: '10:00', 
        end_time: '11:30',
        status: 'confirmed'
      },
      { 
        id: 3, 
        class_name: '고급반',
        date: '2025-04-25',
        start_time: '15:00', 
        end_time: '16:30',
        status: 'cancelled'
      }
    ];
    
    if (myBookings.value.length === 0) {
      $q.notify({
        color: 'warning',
        message: '예약 정보가 없습니다.',
        icon: 'info'
      });
    } else {
      showCheckDialog.value = false;
      showMyBookingsDialog.value = true;
    }
  } catch (error) {
    console.error('예약 확인 실패:', error);
    $q.notify({
      color: 'negative',
      message: error.response?.data?.message || '예약 확인 중 오류가 발생했습니다.',
      icon: 'error'
    });
  }
};

const cancelMyBooking = async (bookingId) => {
  $q.dialog({
    title: '예약 취소',
    message: '예약을 취소하시겠습니까?',
    cancel: true,
    persistent: true
  }).onOk(async () => {
    try {
      // 실제 API 호출 코드
      // const response = await axios.delete(`${API_URL}/bookings/${bookingId}`, {
      //   data: {
      //     password: checkForm.value.password
      //   }
      // });
      
      // 예시 응답 (실제 구현에서는 제거)
      const response = { data: { message: '예약이 취소되었습니다.' } };
      
      $q.notify({
        color: 'positive',
        message: response.data.message,
        icon: 'check_circle'
      });
      
      // 예약 목록 갱신 (실제 구현에서는 API 호출로 다시 가져옴)
      myBookings.value = myBookings.value.map(booking => 
        booking.id === bookingId ? { ...booking, status: 'cancelled' } : booking
      );
    } catch (error) {
      console.error('예약 취소 실패:', error);
      $q.notify({
        color: 'negative',
        message: error.response?.data?.message || '예약 취소 중 오류가 발생했습니다.',
        icon: 'error'
      });
    }
  });
};

// 관리자 관련 함수
const toggleAdminMode = () => {
  if (isAdminMode.value) {
    // 관리자 모드 종료
    isAdminMode.value = false;
    localStorage.removeItem('adminPassword');
  } else {
    // 관리자 모드 시작
    showAdminLoginDialog.value = true;
  }
  
  // 날짜가 선택되어 있으면 다시 로드
  if (selectedDate.value) {
    fetchSchedulesForDate(selectedDate.value);
    if (isAdminMode.value) {
      fetchBookingsForDate();
    }
  }
};

const loginAdmin = () => {
  // 실제 환경에서는 서버에서 비밀번호 검증
  localStorage.setItem('adminPassword', adminPassword.value);
  isAdminMode.value = true;
  showAdminLoginDialog.value = false;
  adminPassword.value = '';
  
  $q.notify({
    color: 'positive',
    message: '관리자 모드로 전환되었습니다.',
    icon: 'check_circle'
  });
  
  if (selectedDate.value) {
    fetchBookingsForDate();
    fetchSchedulesForDate(selectedDate.value); // 관리자 모드로 클래스 목록 다시 가져오기
  }
};

const createClass = async () => {
  try {
    // 실제 API 호출 코드
    // const response = await axios.post(`${API_URL}/classes`, {
    //   admin_password: localStorage.getItem('adminPassword'),
    //   name: classForm.value.name,
    //   max_capacity: classForm.value.max_capacity,
    //   color: classForm.value.color
    // });
    
    // 예시 응답 (실제 구현에서는 제거)
    const response = { data: { message: '클래스가 생성되었습니다.' } };
    
    $q.notify({
      color: 'positive',
      message: response.data.message,
      icon: 'check_circle'
    });
    
    showClassFormDialog.value = false;
    
    // 예시 데이터 추가 (실제 구현에서는 제거)
    classes.value.push({
      id: classes.value.length + 1,
      name: classForm.value.name,
      max_capacity: classForm.value.max_capacity,
      color: classForm.value.color
    });
    
    // 클래스 폼 초기화
    classForm.value = {
      name: '',
      max_capacity: 10,
      color: '#4CAF50'
    };
  } catch (error) {
    console.error('클래스 생성 실패:', error);
    $q.notify({
      color: 'negative',
      message: error.response?.data?.message || '클래스 생성 중 오류가 발생했습니다.',
      icon: 'error'
    });
  }
};

const createSchedule = async () => {
  if (!selectedDate.value) return;
  
  try {
    const apiDateString = formatApiDateString(selectedDate.value);
    
    // 실제 API 호출 코드
    // const response = await axios.post(`${API_URL}/schedules`, {
    //   admin_password: localStorage.getItem('adminPassword'),
    //   class_id: scheduleForm.value.class_id,
    //   date: apiDateString,
    //   start_time: scheduleForm.value.start_time,
    //   end_time: scheduleForm.value.end_time
    // });
    
    // 예시 응답 (실제 구현에서는 제거)
    const response = { data: { message: '스케줄이 생성되었습니다.' } };
    
    $q.notify({
      color: 'positive',
      message: response.data.message,
      icon: 'check_circle'
    });
    
    // 예시 데이터 추가 (실제 구현에서는 제거)
    const selectedClass = classes.value.find(c => c.id === parseInt(scheduleForm.value.class_id));
    if (selectedClass) {
      schedulesForDate.value.push({
        id: Date.now(),  // 임시 ID
        class_id: parseInt(scheduleForm.value.class_id),
        class_name: selectedClass.name,
        date: apiDateString,
        start_time: scheduleForm.value.start_time,
        end_time: scheduleForm.value.end_time,
        available_slots: selectedClass.max_capacity,
        max_capacity: selectedClass.max_capacity,
        color: selectedClass.color
      });
    }
    
    // 스케줄 폼 초기화
    scheduleForm.value = {
      class_id: classes.value.length > 0 ? classes.value[0].id : '',
      start_time: '',
      end_time: ''
    };
    
    // 이벤트 날짜 업데이트
    if (!eventDates.value.includes(selectedDate.value)) {
      eventDates.value.push(selectedDate.value);
    }
  } catch (error) {
    console.error('스케줄 생성 실패:', error);
    $q.notify({
      color: 'negative',
      message: error.response?.data?.message || '스케줄 생성 중 오류가 발생했습니다.',
      icon: 'error'
    });
  }
};

const deleteSchedule = async (scheduleId) => {
  $q.dialog({
    title: '스케줄 삭제',
    message: '이 스케줄을 삭제하시겠습니까?',
    cancel: true,
    persistent: true
  }).onOk(async () => {
    try {
      // 실제 API 호출 코드
      // const response = await axios.delete(`${API_URL}/schedules/${scheduleId}`, {
      //   data: {
      //     admin_password: localStorage.getItem('adminPassword')
      //   }
      // });
      
      // 예시 응답 (실제 구현에서는 제거)
      const response = { data: { message: '스케줄이 삭제되었습니다.' } };
      
      $q.notify({
        color: 'positive',
        message: response.data.message,
        icon: 'check_circle'
      });
      
      // 예시 데이터 제거 (실제 구현에서는 제거)
      schedulesForDate.value = schedulesForDate.value.filter(schedule => schedule.id !== scheduleId);
      
      // 해당 날짜의 스케줄이 모두 삭제되었으면 이벤트 날짜에서도 제거
      if (schedulesForDate.value.length === 0) {
        eventDates.value = eventDates.value.filter(date => date !== selectedDate.value);
      }
    } catch (error) {
      console.error('스케줄 삭제 실패:', error);
      $q.notify({
        color: 'negative',
        message: error.response?.data?.message || '스케줄 삭제 중 오류가 발생했습니다.',
        icon: 'error'
      });
    }
  });
};

const updateBookingStatus = async (bookingId, status) => {
  try {
    // 실제 API 호출 코드
    // const response = await axios.put(`${API_URL}/bookings/${bookingId}`, {
    //   admin_password: localStorage.getItem('adminPassword'),
    //   status: status
    // });
    
    // 예시 응답 (실제 구현에서는 제거)
    const response = { data: { message: `예약 상태가 ${status === 'confirmed' ? '승인' : '거절'}되었습니다.` } };
    
    $q.notify({
      color: 'positive',
      message: response.data.message,
      icon: 'check_circle'
    });
    
    // 예시 데이터 업데이트 (실제 구현에서는 제거)
    bookingsForDate.value = bookingsForDate.value.map(booking => 
      booking.id === bookingId ? { ...booking, status: status } : booking
    );
    
    // 예약 승인/거절에 따른 남은 자리 수 업데이트
    const booking = bookingsForDate.value.find(b => b.id === bookingId);
    if (booking) {
      const schedule = schedulesForDate.value.find(s => s.class_name === booking.class_name);
      if (schedule) {
        if (status === 'confirmed' && booking.status !== 'confirmed') {
          schedule.available_slots -= 1;
        } else if (status === 'cancelled' && booking.status === 'confirmed') {
          schedule.available_slots += 1;
        }
      }
    }
  } catch (error) {
    console.error('예약 상태 변경 실패:', error);
    $q.notify({
      color: 'negative',
      message: error.response?.data?.message || '예약 상태 변경 중 오류가 발생했습니다.',
      icon: 'error'
    });
  }
};

// 초기 데이터 로드
onMounted(() => {
  fetchClasses();
  fetchEventDates();
  
  // 예시 데이터 (실제 구현에서는 제거)
  classes.value = [
    { id: 1, name: '초급반', max_capacity: 5, color: '#4CAF50' },
    { id: 2, name: '중급반', max_capacity: 5, color: '#2196F3' },
    { id: 3, name: '고급반', max_capacity: 4, color: '#F44336' }
  ];
  
  if (classes.value.length > 0) {
    scheduleForm.value.class_id = classes.value[0].id;
  }
  
  // 초기 날짜 선택 시 스케줄 로드
  // 초기에는 자동으로 다이얼로그를 열지 않도록 함
  fetchSchedulesForDate(selectedDate.value, false);
});

// 날짜 선택 시 자동으로 스케줄 로드
watchEffect(() => {
  if (selectedDate.value) {
    onDateSelected(selectedDate.value);
  }
});
</script><style>
/* QDate 커스텀 스타일 */
.q-date__calendar-item .q-date__event {
  position: center;
}

.q-date__calendar-item .q-date__event::after {
  content: '';
  position: absolute;
  bottom: 2px;
  left: 50%;
  transform: translateX(-50%);
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background-color: var(--q-primary);
}

/* 카드 스타일 */
.q-card {
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  transition: box-shadow 0.3s ease;
}

.q-card:hover {
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
}

/* 리스트 아이템 스타일 */
.q-item {
  border-left: 3px solid transparent;
  transition: background-color 0.2s;
}

.q-item:hover {
  background-color: rgba(0, 0, 0, 0.03);
}

/* 버튼 스타일 */
.q-btn {
  letter-spacing: 0.5px;
  font-weight: 500;
}

/* 스케줄 상태 색상 */
.bg-amber-1 {
  background-color: rgba(255, 193, 7, 0.1) !important;
}

/* 모바일 최적화 */
@media (max-width: 599px) {
  .q-date {
    width: 100%;
  }
  .q-btn {
    min-height: 36px;
  }
}

/* 태블릿 스타일 */
@media (min-width: 600px) and (max-width: 1023px) {
  .q-date {
    width: auto;
  }
}

/* 스크롤 스타일 */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

::-webkit-scrollbar-track {
  background: #f1f1f1;
}

::-webkit-scrollbar-thumb {
  background: #ccc;
  border-radius: 4px;
}

::-webkit-scrollbar-thumb:hover {
  background: #aaa;
}

/* 관리자 버튼 스타일 - 모바일 친화적 */
.admin-button-wrapper {
  padding: 4px;
  border-radius: 50%;
  touch-action: manipulation;
}

.admin-icon {
  opacity: 0.3;
}

/* 모바일 최적화 */
@media (max-width: 599px) {
  /* 다른 스타일들... */
  
  /* 모바일에서 관리자 아이콘 더 숨기기 */
  .admin-icon {
    opacity: 0.2;
  }
}
</style>