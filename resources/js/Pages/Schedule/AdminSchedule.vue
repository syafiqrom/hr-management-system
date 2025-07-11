<script setup>
import { ref, computed, onMounted, watch } from 'vue';
import dayjs from 'dayjs';
import Card from '@/Components/Card.vue';
import AuthenticatedLayout from '@/Layouts/AuthenticatedLayout.vue';
import { Head } from '@inertiajs/vue3';
import axios from 'axios';
import ScheduleTabs from '@/Components/Tabs/ScheduleTabs.vue';
import FlexButton from '@/Components/FlexButton.vue';

const props = defineProps({
  staffList: {
    type: Array,
    default: () => [],
  },
  leaveList: {
    type: Array,
    default: () => [],
  }
});

console.log('staffList:', props.staffList);

const shiftNames = ['Morning', 'Night'];
const shiftTimes = [
  { label: 'Morning', start: '06:00', end: '15:00' },
  { label: 'Night', start: '15:00', end: '00:00' }
];

// State for the selected week (start on Monday)
const today = dayjs();
const thisMonday = today.startOf('week').add(1, 'day');
const currentMonday = ref(today.isBefore(thisMonday, 'day') ? thisMonday : thisMonday.add(7, 'day'));

// State for the selected day and shift
const selectedDay = ref(null);
const selectedShiftIdx = ref(null);

// Loading state for validation
const isValidating = ref(false);

// Assignments: Each day holds two shifts, each shift holds staffId or null
const assignments = ref({});

// Local state for the modal selection
const selectedStaffId = ref('');

// Fetch assignments for the current week from backend
const isSubmitted = ref(false);

async function fetchAssignments() {
  const weekStart = currentMonday.value.format('YYYY-MM-DD');
  const { data } = await axios.get('/schedule/week', { params: { week_start: weekStart } });
  assignments.value = data.assignments || {};
  isSubmitted.value = data.submitted || false;
}

// Fetch assignments on page load and when week changes
onMounted(fetchAssignments);
watch(currentMonday, fetchAssignments);

// Calculate the days for the selected week
const weekDays = computed(() => {
  return Array.from({ length: 7 }, (_, i) => currentMonday.value.add(i, 'day'));
});

// Open modal for a specific day and shift
function openDayModal(day, shiftIdx) {
  selectedDay.value = day;
  selectedShiftIdx.value = shiftIdx;
  if (!assignments.value[day.format('YYYY-MM-DD')]) {
    assignments.value[day.format('YYYY-MM-DD')] = [null, null];
  }
  selectedStaffId.value = assignments.value[day.format('YYYY-MM-DD')][shiftIdx] || '';
}

// Close modal
function closeDayModal() {
  if (isValidating.value) return;
  selectedDay.value = null;
  selectedShiftIdx.value = null;
}

// Assign staff to a shift with loading animation and backend call
async function assignStaff() {
  isValidating.value = true;
  await new Promise(resolve => setTimeout(resolve, 1000));

  const staffId = selectedStaffId.value;
  const dayKey = selectedDay.value.format('YYYY-MM-DD');
  const weekStart = currentMonday.value.format('YYYY-MM-DD');

  // 1. Check if staff is on leave that day
  // leaveList should be an array of { employee_id, date }
  if (props.leaveList && props.leaveList.some(l => l.employee_id == staffId && l.date === dayKey)) {
    isValidating.value = false;
    alert('This staff is on leave for the selected day. Please pick another staff.');
    return;
  }

  // 2. Check if staff is already assigned to another shift on the same day
  const assignmentsForDay = assignments.value[dayKey] || [null, null];
  if (assignmentsForDay.includes(staffId)) {
    isValidating.value = false;
    alert('This staff is already assigned to another shift on this day. Please pick another staff.');
    return;
  }

  // 3. Check if staff is assigned to more than 6 days in the week
  let daysAssigned = 0;
  for (const [date, shifts] of Object.entries(assignments.value)) {
    if (date >= weekStart && date <= dayjs(weekStart).add(6, 'day').format('YYYY-MM-DD')) {
      if (shifts.includes(staffId)) daysAssigned++;
    }
  }
  if (daysAssigned >= 6) {
    isValidating.value = false;
    alert('This staff is already assigned to 6 days in this week. Please pick another staff.');
    return;
  }

  // If all checks pass, proceed to save
  await axios.post('/schedule/assign', {
    employee_id: staffId,
    shift_type: shiftNames[selectedShiftIdx.value].toLowerCase(),
    week_start: weekStart,
    day: dayKey,
  });
  await fetchAssignments();
  isValidating.value = false;
  closeDayModal();
}

// Navigate to previous week
function prevWeek() {
  currentMonday.value = currentMonday.value.subtract(1, 'week');
  fetchAssignments();
}

// Navigate to next week
function nextWeek() {
  currentMonday.value = currentMonday.value.add(1, 'week');
  fetchAssignments();
}

// Submit the schedule
async function submitSchedule() {
  // //If today is not Sunday (the day before the selected week starts), show an alert
  // //"Supervisor can only submit the schedule one day before the week starts (on Sunday)."
  // const today = dayjs();
  // const nextMonday = currentMonday.value;
  // const sundayBeforeWeek = nextMonday.subtract(1, 'day');
  // //Only allow submit if today is the Sunday before the selected week
  // if (!today.isSame(sundayBeforeWeek, 'day')) {
  //   alert('Supervisor can only submit the schedule one day before the week starts (on Sunday).');
  //   return;
  // }
  // alert('Weekly schedule submitted successfully!');
  // Move to next week
  await axios.post('/schedule/submit-week', {
    week_start: currentMonday.value.format('YYYY-MM-DD'),
  });
  await fetchAssignments();
  alert('Weekly schedule submitted successfully!');
  currentMonday.value = currentMonday.value.add(7, 'day');
}

// Get staff name for a shift
function getStaffName(day, shiftIdx) {
  const staffId = assignments.value[day.format('YYYY-MM-DD')]?.[shiftIdx];
  const staff = props.staffList.find(s => s.id == staffId);
  return staff ? staff.name : '';
}

// Check if a shift is assigned
function isShiftAssigned(day, shiftIdx) {
  return !!assignments.value[day.format('YYYY-MM-DD')]?.[shiftIdx];
}

// Reset all assignments for the current week
async function resetAssignments() {
  if (!confirm('Are you sure you want to reset all assignments for this week?')) return;
  // Optionally, call backend to delete all assignments for this week
  await axios.post('/schedule/reset', {
    week_start: currentMonday.value.format('YYYY-MM-DD'),
  });
  assignments.value = {};
  await fetchAssignments();
}
</script>

<template>
  <Head title="Weekly Schedule" />
  <AuthenticatedLayout>
    <template #tabs>
      <ScheduleTabs />
    </template>
    <div class="py-8">
      <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
        <Card>
          <div class="flex justify-between items-center mb-6">
            <FlexButton :text="'Previous'" @click="prevWeek" />
            <span class="font-semibold text-lg">{{ currentMonday.format('MMM D, YYYY') }} - {{ currentMonday.add(6, 'day').format('MMM D, YYYY') }}</span>
            <FlexButton :text="'Next'" @click="nextWeek" />
          </div>
        </Card>
        <Card class="mt-8">
          <div class="px-6 pb-6">
            <table class="w-full border-separate border-spacing-y-2">
              <thead>
                <tr>
                  <th class="bg-gray-100 text-base font-bold text-gray-700 py-4 text-left">Day</th>
                  <th class="bg-gray-100 text-base font-bold text-gray-700 py-4 text-center">Morning<br><span class="text-xs text-gray-400">6:00 AM - 3:00 PM</span></th>
                  <th class="bg-gray-100 text-base font-bold text-gray-700 py-4 text-center">Night<br><span class="text-xs text-gray-400">3:00 PM - 12:00 AM</span></th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(day, dayIdx) in weekDays" :key="dayIdx" class="border-b last:border-b-0">
                  <td class="py-4 px-4 text-lg font-bold text-gray-700 w-40 align-top">
                    <div class="flex flex-col">
                      <span class="text-xs font-semibold text-gray-400 uppercase">{{ day.format('ddd') }}</span>
                      <span class="text-2xl font-extrabold">{{ day.format('D') }}</span>
                    </div>
                  </td>
                  <td v-for="shiftIdx in [0,1]" :key="shiftIdx" class="py-4 px-2 align-top">
                    <div
                      :class="[
                        'w-full h-16 flex items-center justify-center rounded-lg transition text-base font-medium border-2',
                        isShiftAssigned(day, shiftIdx)
                          ? 'bg-green-200 text-green-900 border-green-300 shadow'
                          : 'bg-gray-100 text-gray-400 border-gray-200 hover:bg-purple-100 hover:text-purple-700 hover:border-purple-300',
                        isSubmitted ? 'opacity-50 cursor-not-allowed' : 'cursor-pointer'
                      ]"
                      @click="isSubmitted ? null : openDayModal(day, shiftIdx)"
                    >
                      <span v-if="isShiftAssigned(day, shiftIdx)">
                        {{ getStaffName(day, shiftIdx) }}
                      </span>
                      <span v-else>
                        {{ shiftNames[shiftIdx] }}
                      </span>
                    </div>
                  </td>
                </tr>
              </tbody>
            </table>
            <div class="flex justify-end mt-8 space-x-4">
              <FlexButton :text="'Reset'" @click="resetAssignments" :class="'bg-red-600 text-white px-8 py-2 rounded-lg font-semibold text-base'" :disabled="isSubmitted" />
              <FlexButton
                v-if="!isSubmitted"
                :text="'Submit Weekly Schedule'"
                @click="submitSchedule"
                :class="'bg-purple-700 text-white px-8 py-2 rounded-lg font-semibold text-base'"
                :disabled="isSubmitted"
              />
              <FlexButton
                v-else
                :text="'Assign Task'"
                @click="$inertia.visit(route('schedule.assign-task'))"
                :class="'bg-green-700 text-white px-8 py-2 rounded-lg font-semibold text-base'"
              />
            </div>
          </div>
        </Card>
        <!-- Modal -->
        <transition name="fade">
          <div v-if="selectedDay !== null && selectedShiftIdx !== null && !isSubmitted" class="fixed inset-0 bg-black bg-opacity-40 flex items-center justify-center z-50">
            <div class="bg-white rounded-2xl shadow-2xl p-8 w-full max-w-md relative animate-fadeIn">
              <button @click="closeDayModal" class="absolute top-3 right-3 text-gray-400 hover:text-gray-700 text-xl font-bold" :disabled="isValidating">&times;</button>
              <h2 class="text-2xl font-bold mb-6 text-gray-800 text-center">
                Assign {{ shiftNames[selectedShiftIdx] }} Shift<br>
                <span class="text-base font-medium text-gray-500">for {{ selectedDay.format('ddd, MMM D') }}</span>
              </h2>
              <div v-if="isValidating" class="flex flex-col items-center justify-center py-8">
                <svg class="animate-spin h-10 w-10 text-purple-500 mb-4" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                  <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                  <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v4a4 4 0 00-4 4H4z"></path>
                </svg>
                <span class="text-purple-600 font-semibold text-lg">Validating staff...</span>
              </div>
              <div v-else class="mb-6">
                <label class="block text-sm font-semibold mb-2 text-gray-700">Select Staff</label>
                <select
                  v-model="selectedStaffId"
                  class="border-2 border-purple-300 focus:border-purple-500 focus:ring-2 focus:ring-purple-200 rounded-lg px-3 py-2 w-full text-base transition shadow-sm outline-none"
                  :disabled="isValidating"
                  autofocus
                >
                  <option value="">Select Staff</option>
                  <option v-for="staff in props.staffList" :key="staff.id" :value="staff.id">{{ staff.name }}</option>
                </select>
                <FlexButton
                  :text="'Assign'"
                  @click="assignStaff"
                  :class="'w-full bg-purple-600 text-white font-semibold py-2 rounded-lg mt-4'"
                  :disabled="isValidating || !selectedStaffId"
                />
              </div>
              <FlexButton @click="closeDayModal" :text="'Cancel'" :class="'w-full bg-gray-200 hover:bg-gray-300 text-gray-700 font-semibold py-2 rounded-lg transition'" :disabled="isValidating" />
            </div>
          </div>
        </transition>
      </div>
    </div>
  </AuthenticatedLayout>
</template>

<style scoped>
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.2s;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
.animate-fadeIn {
  animation: fadeIn 0.3s;
}
@keyframes fadeIn {
  from { opacity: 0; transform: scale(0.96); }
  to { opacity: 1; transform: scale(1); }
}
</style> 