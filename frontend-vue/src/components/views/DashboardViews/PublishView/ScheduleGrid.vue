<script setup lang="ts">
  import { ref, onMounted, onUnmounted } from 'vue';
  import Dialog from 'primevue/dialog';
  import Button from 'primevue/button';
  import Textarea from 'primevue/textarea';
  import MultiSelect from 'primevue/multiselect';
  import { format } from 'date-fns';
  import { useToast } from 'primevue/usetoast';

  import connectionsDataStore from '@/utils/connectionsDataStore';
  // import threadsIcon from '@/assets/dashboardSocialIcons/black/threadsBlack.svg';
  // import xIcon from '@/assets/dashboardSocialIcons/black/xBlack.svg';
  // import blueskyIcon from '@/assets/dashboardSocialIcons/black/blueskyBlack.svg';
  // import tiktokIcon from '@/assets/dashboardSocialIcons/black/tiktokBlack.svg';
  // import youtubeIcon from '@/assets/dashboardSocialIcons/black/youtubeBlack.svg';
  // import instagramIcon from '@/assets/dashboardSocialIcons/black/instagramBlack.svg';

  import appSettings from '@/utils/appSettings';
  import scheduledPostsStore from '@/utils/scheduledPostsStore';
  import publishViewDataStore from '@/utils/publishViewDataStore';
  import { SquarePlus } from 'lucide-vue-next';

  import { useThemeStore } from '@/utils/themeStore';
  const themeStore = useThemeStore();

  const scrollContainer = ref<HTMLDivElement | null>(null); // Ref for the scroll container

  const toast = useToast();

  interface TimeSlot {
    time: string;
    display: string;
    isHourStart: boolean;
    isLastSlotOfHour: boolean;
  }

  type WeekDay =
    | 'Sunday'
    | 'Monday'
    | 'Tuesday'
    | 'Wednesday'
    | 'Thursday'
    | 'Friday'
    | 'Saturday';

  const emit = defineEmits<{
    (
      e: 'schedulePost',
      content: string,
      scheduledTime: Date,
      selectedPlatforms: string[]
    ): void;
    (e: 'timeSlotClick', dateTime: Date): void;
    (e: 'editPost', post: any): void;
  }>();

  const currentMonth = ref('');
  const timeSlots = ref<TimeSlot[]>(generateTimeSlots());
  const selectedSlot = ref<{ time: string; day: string } | null>(null);
  const showScheduleDialog = ref(false);
  const content = ref('');
  const selectedPlatforms = ref<string[]>([]);

  const weekDays = [
    'Sunday',
    'Monday',
    'Tuesday',
    'Wednesday',
    'Thursday',
    'Friday',
    'Saturday',
  ] as const;

  function generateTimeSlots() {
    const slots = [];
    for (let hour = 0; hour < 24; hour++) {
      const isPastNoon = hour >= 12;
      const displayHour = hour === 0 ? 12 : hour > 12 ? hour - 12 : hour;
      const period = isPastNoon ? 'pm' : 'am';
      const display = `${displayHour}:00${period}`;

      slots.push({
        time: `${hour.toString().padStart(2, '0')}:00`,
        display,
        isHourStart: true,
        isLastSlotOfHour: true,
      });
    }

    return slots;
  }

  function getWeekDates() {
    const today = new Date();
    const currentDay = today.getDay();
    const diff = currentDay;
    const lastSunday = new Date(
      today.setDate(
        today.getDate() - diff + publishViewDataStore.weekOffset.value * 7
      )
    );

    const monthNames = [
      'January',
      'February',
      'March',
      'April',
      'May',
      'June',
      'July',
      'August',
      'September',
      'October',
      'November',
      'December',
    ];
    const firstDayOfWeek = new Date(lastSunday);
    currentMonth.value = `${monthNames[firstDayOfWeek.getMonth()]} ${firstDayOfWeek.getFullYear()}`;

    return weekDays.map((day, index) => {
      const date = new Date(lastSunday);
      date.setDate(lastSunday.getDate() + index);
      return `${day} ${date.getDate()}`;
    });
  }

  const weekDatesWithDays = ref(getWeekDates());

  function getDaysInMonth(year: number, month: number): number {
    return new Date(year, month + 1, 0).getDate();
  }

  function getMonthDates() {
    const today = new Date();
    today.setMonth(today.getMonth() + publishViewDataStore.monthOffset.value);
    const year = today.getFullYear();
    const month = today.getMonth();

    const firstDay = new Date(year, month, 1);
    const startingDay = firstDay.getDay();
    const daysInMonth = getDaysInMonth(year, month);

    const monthDates = [];
    let week = [];

    // Add empty cells for days before the first of the month
    for (let i = 0; i < startingDay; i++) {
      week.push({ date: null, isCurrentMonth: false });
    }

    // Add days of the month
    for (let day = 1; day <= daysInMonth; day++) {
      week.push({
        date: new Date(year, month, day),
        isCurrentMonth: true,
        hasEvents: scheduledPostsStore.scheduledPosts.value.some((post) => {
          const postDate = new Date(post.scheduledTime);

          // Compare year, month, day
          const isSameDay =
            postDate.getFullYear() === year &&
            postDate.getMonth() === month &&
            postDate.getDate() === day;

          return isSameDay;
        }),
      });

      if ((startingDay + day) % 7 === 0 || day === daysInMonth) {
        monthDates.push(week);
        week = [];
      }
    }

    // Fill in remaining days of the last week if needed
    if (week.length > 0) {
      const remainingDays = 7 - week.length;
      for (let i = 0; i < remainingDays; i++) {
        week.push({ date: null, isCurrentMonth: false });
      }
      monthDates.push(week);
    }

    const monthNames = [
      'January',
      'February',
      'March',
      'April',
      'May',
      'June',
      'July',
      'August',
      'September',
      'October',
      'November',
      'December',
    ];
    currentMonth.value = `${monthNames[month]} ${year}`;

    return monthDates;
  }

  const monthDates = ref(getMonthDates());

  const currentTime = ref(new Date());
  const timeUpdateInterval = ref<ReturnType<typeof setInterval> | null>(null);

  const formatTime = (date: Date | string) => {
    const dateObj = typeof date === 'string' ? new Date(date) : date;
    return format(dateObj, 'h:mm a');
  };

  const updateScrollPosition = () => {
    if (scrollContainer.value) {
      publishViewDataStore.savedScrollPosition.value =
        scrollContainer.value.scrollTop;
    }
  };

  const scrollToCurrentHour = () => {
    if (scrollContainer.value) {
      const currentHour = new Date().getHours();
      const hourId = `hour-${currentHour.toString().padStart(2, '0')}`;
      const hourElement = document.getElementById(hourId);

      if (hourElement) {
        const elementPosition = hourElement.offsetTop;
        // Scroll a bit above the current hour for better visibility
        scrollContainer.value.scrollTop = Math.max(0, elementPosition - 100);
      }
    }
  };

  onMounted(async () => {
    if (scrollContainer.value) {
      scrollContainer.value.scrollTop =
        publishViewDataStore.savedScrollPosition.value || 0;
    }

    await connectionsDataStore.getAllAccounts();

    if (publishViewDataStore.savedScrollPosition.value === null) {
      setTimeout(scrollToCurrentHour, 300);
    }

    // Update time immediately and log it
    currentTime.value = new Date();

    // Then update every minute instead of every second
    timeUpdateInterval.value = setInterval(() => {
      currentTime.value = new Date();
    }, 60000); // Update every minute

    scrollContainer.value?.addEventListener('scroll', updateScrollPosition);
  });

  onUnmounted(() => {
    if (timeUpdateInterval.value) {
      clearInterval(timeUpdateInterval.value);
    }
    if (scrollContainer.value) {
      publishViewDataStore.savedScrollPosition.value =
        scrollContainer.value.scrollTop;
      scrollContainer.value.removeEventListener('scroll', updateScrollPosition);
    }
  });

  function navigate(direction: number) {
    if (publishViewDataStore.viewType.value === 'week') {
      publishViewDataStore.weekOffset.value += direction;
      weekDatesWithDays.value = getWeekDates();
    } else {
      publishViewDataStore.monthOffset.value += direction;
      monthDates.value = getMonthDates();
    }
  }

  function resetToToday() {
    publishViewDataStore.weekOffset.value = 0;
    publishViewDataStore.monthOffset.value = 0;
    weekDatesWithDays.value = getWeekDates();
    monthDates.value = getMonthDates();
  }

  function handleTimeSlotClick(time: string, day: WeekDay) {
    const today = new Date();
    const selectedDay = weekDays.indexOf(day);
    const [hours, minutes] = time.split(':').map(Number);
    let targetDate = new Date(today);
    const currentDay = today.getDay();

    if (publishViewDataStore.weekOffset.value < 0) {
      toast.add({
        severity: 'error',
        summary: 'Invalid Time Selection',
        detail:
          'Cannot schedule posts for past dates. Please select a future time slot.',
        life: 3000,
      });
      return;
    }

    if (publishViewDataStore.weekOffset.value > 0) {
      const daysToAdd =
        7 * publishViewDataStore.weekOffset.value + selectedDay - currentDay;
      targetDate.setDate(today.getDate() + daysToAdd);
      targetDate.setHours(hours, minutes, 0, 0);
      emit('timeSlotClick', targetDate);
      return;
    }

    if (selectedDay < currentDay) {
      toast.add({
        severity: 'error',
        summary: 'Invalid Time Selection',
        detail:
          'Cannot schedule posts for past dates. Please select a future time slot.',
        life: 3000,
      });
      return;
    }

    if (selectedDay === currentDay) {
      if (hours < today.getHours()) {
        toast.add({
          severity: 'error',
          summary: 'Invalid Time Selection',
          detail:
            'Cannot schedule posts for past dates. Please select a future time slot.',
          life: 3000,
        });
        return;
      }
    }

    const daysToAdd = selectedDay - currentDay;
    targetDate.setDate(today.getDate() + daysToAdd);
    targetDate.setHours(hours, minutes, 0, 0);
    emit('timeSlotClick', targetDate);
  }

  function handleMonthDayClick(date: Date | null) {
    if (!date) return;

    selectedSlot.value = {
      day: weekDays[date.getDay()],
      time: '09:00',
    };
    content.value = '';
    selectedPlatforms.value = [];
    showScheduleDialog.value = true;
  }

  function handleSchedulePost() {
    emit('schedulePost', content.value, new Date(), selectedPlatforms.value);
    showScheduleDialog.value = false;
    content.value = '';
    selectedPlatforms.value = [];
  }

  function isToday(date: Date | null): boolean {
    if (!date) return false;
    const today = new Date();
    return (
      date.getDate() === today.getDate() &&
      date.getMonth() === today.getMonth() &&
      date.getFullYear() === today.getFullYear()
    );
  }

  function isTodayColumn(dayWithDate: string): boolean {
    const today = new Date();
    const [day, date] = dayWithDate.split(' ');
    return (
      day === weekDays[today.getDay()] && parseInt(date) === today.getDate()
    );
  }

  function getScheduledPostsForSlot(day: WeekDay, time: string) {
    if (
      !scheduledPostsStore.scheduledPosts.value ||
      scheduledPostsStore.scheduledPosts.value.length === 0
    ) {
      return [];
    }

    // Get current week's date range based on weekOffset
    const currentDate = new Date();
    const startOfWeek = new Date(currentDate);
    startOfWeek.setDate(
      currentDate.getDate() + publishViewDataStore.weekOffset.value * 7
    );
    startOfWeek.setDate(startOfWeek.getDate() - startOfWeek.getDay()); // Set to Sunday

    const targetDayIndex = weekDays.indexOf(day);
    const targetDate = new Date(startOfWeek);
    targetDate.setDate(startOfWeek.getDate() + targetDayIndex);

    const [hour] = time.split(':').map(Number);

    if (scheduledPostsStore.scheduledPosts.value.length === 0) {
    }
    const filteredPosts = scheduledPostsStore.scheduledPosts.value.filter(
      (post) => {
        const postDate = new Date(post.scheduledTime);

        // Compare year, month, day
        const isSameDay =
          postDate.getFullYear() === targetDate.getFullYear() &&
          postDate.getMonth() === targetDate.getMonth() &&
          postDate.getDate() === targetDate.getDate();

        // Check if post is within this hour slot
        const isSameHour = postDate.getHours() === hour;

        return isSameDay && isSameHour;
      }
    );

    return filteredPosts;
  }

  function isTimeSlotInPast(time: string, day: WeekDay): boolean {
    const today = new Date();
    const selectedDay = weekDays.indexOf(day);
    const currentDay = today.getDay();
    const [hours, _] = time.split(':').map(Number);

    if (publishViewDataStore.weekOffset.value < 0) return true;
    if (selectedDay < currentDay && publishViewDataStore.weekOffset.value === 0)
      return true;
    if (
      selectedDay === currentDay &&
      publishViewDataStore.weekOffset.value === 0
    ) {
      if (hours < today.getHours()) return true;
    }
    return false;
  }
</script>

<template>
  <div class="schedule-grid mt-8 rounded-lg py-4">
    <div class="mb-4 flex items-center">
      <div class="flex items-center gap-2">
        <button
          class="flex h-8 w-8 items-center justify-center rounded-lg border text-lg hover:bg-gray-50 dark:border-[#313131] dark:hover:bg-[#313131]/80"
          @click="navigate(-1)"
        >
          &lt;
        </button>
        <span class="mx-2 font-medium">{{ currentMonth }}</span>

        <button
          class="flex h-8 w-8 items-center justify-center rounded-lg border text-lg hover:bg-gray-50 dark:border-[#313131] dark:hover:bg-[#313131]/80"
          @click="navigate(1)"
        >
          &gt;
        </button>
      </div>
      <div class="ml-4 flex items-center">
        <button
          class="flex items-center justify-center rounded-lg border px-3 py-1 text-sm hover:bg-gray-50 dark:border-[#313131] dark:hover:bg-[#313131]/80"
          @click="resetToToday"
        >
          Today
        </button>
      </div>
    </div>

    <!-- Weekly View -->
    <div
      v-if="publishViewDataStore.viewType.value === 'week'"
      class="relative h-auto overflow-hidden"
    >
      <div class="relative rounded-[10px] border border-[transparent]">
        <table class="box-border w-full min-w-full table-fixed">
          <thead>
            <tr>
              <th class="w-[70px] border-b border-r border-[transparent]"></th>
              <template
                v-for="dayWithDate in weekDatesWithDays"
                :key="dayWithDate"
              >
                <th
                  class="border-b border-r border-[transparent] p-2 text-center text-[14px] font-medium"
                  :class="isTodayColumn(dayWithDate) ? 'bg-[#0000000b]' : ''"
                >
                  {{ dayWithDate }}
                </th>
              </template>
              <th class="w-[10px]"></th>
            </tr>
          </thead>
        </table>
      </div>

      <div
        ref="scrollContainer"
        class="scrollbar-thin border-1 relative h-[calc(100vh-300px)] max-h-fit min-h-[500px] overflow-y-auto overflow-x-hidden rounded-[10px] border border-[#c1c1c1] bg-[white] dark:bg-[#464646]"
      >
        <div class="relative">
          <table class="w-full min-w-full table-fixed border-collapse">
            <tbody class="relative">
              <tr
                v-for="slot in timeSlots"
                :key="slot.time"
                class="relative"
                :class="!slot.isHourStart ? 'border-b' : 'border-b-dotted'"
                :id="'hour-' + slot.time.split(':')[0]"
              >
                <td
                  class="w-[70px] border-r text-center text-[12px] dark:border-[#9c9c9c82]"
                >
                  <div class="h-[50px]">
                    {{ slot.isHourStart ? slot.display : '' }}
                  </div>
                </td>
                <template
                  v-for="(day, index) in weekDays"
                  :key="`${day}-${slot.time}`"
                >
                  <td
                    class="group border-r p-[5px] text-center align-top last:border-r-0 dark:border-[#9c9c9c82]"
                    :class="[
                      isTodayColumn(weekDatesWithDays[index]) ? '' : '',
                      isTimeSlotInPast(slot.time, day)
                        ? 'bg-[#00000007] dark:bg-[#5a5a5a]'
                        : 'hover:bg-gray-50 dark:hover:bg-[#5a5a5a]',
                    ]"
                  >
                    <div
                      class="flex h-min min-h-[50px] flex-grow-0 flex-col items-end gap-2"
                    >
                      <div
                        v-for="post in getScheduledPostsForSlot(day, slot.time)"
                        :key="post.id"
                        id="post-note"
                        class="z-50 flex min-h-[50px] w-full max-w-[85%] flex-1 cursor-pointer rounded-l-[5px] bg-[white] text-[12px] shadow-md transition-all hover:max-w-[100%] dark:bg-[#292929]"
                        v-tooltip.top="post.content"
                        @click="
                          !isTimeSlotInPast(slot.time, day) &&
                            (post.status === 'draft' ||
                              post.status === 'scheduled') &&
                            $emit('editPost', post)
                        "
                      >
                        <div
                          class="w-[3px] flex-shrink-0 rounded-l-[5px] pl-[5px]"
                          :class="
                            post.status === 'scheduled' ||
                            post.status === 'processing'
                              ? 'bg-yellow-500'
                              : post.status === 'published'
                                ? 'bg-green'
                                : post.status === 'partially_published' ||
                                    post.status === 'processing'
                                  ? 'bg-orange-500'
                                  : post.status === 'failed'
                                    ? 'bg-red-500'
                                    : post.status === 'draft'
                                      ? 'bg-gray-500'
                                      : 'bg-red-500'
                          "
                        ></div>
                        <!-- <div class="icons flex p-[1px]">
                          <div
                            v-for="platform in post.platforms"
                            :key="platform"
                          >
                            <div v-if="platform == 'threads'">
                              <img
                                :src="threadsIcon"
                                class="test h-[10px] w-[10px] rounded-full object-cover"
                                alt="threads icon"
                              />
                            </div>
                            <div v-if="platform == 'twitter'">
                              <img
                                :src="xIcon"
                                class="h-[10px] w-[10px] rounded-full object-cover"
                                alt="twitter icon"
                              />
                            </div>
                            <div v-if="platform == 'bluesky'">
                              <img
                                :src="blueskyIcon"
                                class="h-[10px] w-[10px] rounded-full object-cover"
                                alt="bluesky icon"
                              />
                            </div>
                            <div v-if="platform == 'tiktok'">
                              <img
                                :src="tiktokIcon"
                                class="h-[10px] w-[10px] rounded-full object-cover"
                                alt="tiktok icon"
                              />
                            </div>
                            <div v-if="platform == 'youtube'">
                              <img
                                :src="youtubeIcon"
                                class="h-[10px] w-[10px] rounded-full object-cover"
                                alt="youtube icon"
                              />
                            </div>
                            <div v-if="platform == 'instagram'">
                              <img
                                :src="instagramIcon"
                                class="h-[10px] w-[10px] rounded-full object-cover"
                                alt="instagram icon"
                              />
                            </div>
                          </div>
                        </div> -->

                        <div class="flex flex-1 flex-col">
                          <div class="flex w-full items-center justify-end">
                            <p
                              class="flex-0 line-clamp-1 flex-shrink-0 pl-[5px] text-[10px] text-[black]"
                            >
                              {{ formatTime(post.scheduledTime) }}
                            </p>
                          </div>
                          <p
                            class="shrink-1 line-clamp-1 w-full flex-1 pl-[2px] text-start text-[10px] text-[#525252]"
                          >
                            {{ post.content }}
                          </p>
                        </div>
                      </div>
                      <div
                        class="add flex h-[50px] w-full items-center justify-center rounded-[5px] border-[1px] border-[transparent] bg-[white] opacity-0 transition-all duration-200 dark:bg-[#5a5a5a]"
                        :class="[
                          isTimeSlotInPast(slot.time, day)
                            ? 'bg-[transparent] opacity-0'
                            : 'add-post cursor-pointer group-hover:border-[#0000004d] group-hover:opacity-30 dark:group-hover:border-[#ffffff4d]',
                        ]"
                        @click="
                          !isTimeSlotInPast(slot.time, day) &&
                            handleTimeSlotClick(slot.time, day)
                        "
                      >
                        <SquarePlus
                          :color="
                            themeStore.currentTheme === 'light'
                              ? '#00000082'
                              : '#ececec82'
                          "
                        />
                      </div>
                    </div>
                  </td>
                </template>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- Monthly View -->
    <div v-else class="relative h-auto overflow-hidden">
      <div class="relative rounded-[10px] border border-[#c1c1c1]">
        <table class="box-border w-full min-w-full table-fixed border-collapse">
          <thead>
            <tr>
              <th
                v-for="day in weekDays"
                :key="day"
                class="border-b border-r border-[#c1c1c1] p-2 text-center text-[14px] font-medium last:border-r-0"
              >
                {{ day.slice(0, 3) }}
              </th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="(week, weekIndex) in monthDates"
              :key="weekIndex"
              class="border-b border-[#c1c1c1] last:border-b-0"
            >
              <td
                v-for="(day, dayIndex) in week"
                :key="dayIndex"
                class="relative h-24 border-l border-r border-[#c1c1c1] p-2 first:border-l-0 last:border-r-0"
                :class="{
                  'bg-gray-50': !day.isCurrentMonth,
                  'bg-[#0000000b]': day.isCurrentMonth && isToday(day.date),
                  'cursor-pointer hover:bg-gray-50': day.isCurrentMonth,
                }"
                @click="day.isCurrentMonth && handleMonthDayClick(day.date)"
              >
                <span
                  :class="{
                    'text-gray-400': !day.isCurrentMonth,
                    'font-medium': day.isCurrentMonth,
                    'text-blue-600': day.isCurrentMonth && isToday(day.date),
                  }"
                >
                  {{ day.date?.getDate() }}
                </span>
                <div
                  v-if="day.hasEvents"
                  class="mt-2 h-2 w-2 rounded-full bg-blue-500"
                ></div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Schedule Dialog -->
    <Dialog
      v-model:visible="showScheduleDialog"
      modal
      header="Schedule Post"
      :style="{ width: '90%', maxWidth: '500px' }"
    >
      <div class="p-4">
        <p class="mb-4">
          Scheduling for {{ selectedSlot?.day }} at {{ selectedSlot?.time }}
        </p>
        <div class="mb-4">
          <label class="mb-1 block text-sm font-medium text-gray-700"
            >Content</label
          >
          <Textarea
            v-model="content"
            rows="4"
            class="w-full rounded border p-2"
            placeholder="What's on your mind?"
          ></Textarea>
        </div>
        <div class="mb-4">
          <label class="mb-1 block text-sm font-medium text-gray-700"
            >Platforms</label
          >
          <MultiSelect
            v-model="selectedPlatforms"
            :options="appSettings.platformOptions"
            optionLabel="label"
            placeholder="Select platforms"
            class="w-full"
          />
        </div>
        <div class="flex justify-end gap-2">
          <Button
            label="Cancel"
            @click="showScheduleDialog = false"
            class="p-button-text"
          ></Button>
          <Button label="Schedule" @click="handleSchedulePost"></Button>
        </div>
      </div>
    </Dialog>
  </div>
</template>

<style scoped>
  .scrollbar-thin::-webkit-scrollbar {
    background: #f1f1f1;
    width: 10px;
    height: 10px;
    border-top-right-radius: 10px;
    border-bottom-right-radius: 10px;
  }

  .scrollbar-thin::-webkit-scrollbar-thumb {
    background: rgb(140, 140, 140);
    border-radius: 10px;
    border: 2px solid #f1f1f1; /* This creates padding around the thumb */
    background-clip: padding-box; /* This ensures the background doesn't show through the border */
  }

  .scrollbar-thin::-webkit-scrollbar-track {
    background: #f1f1f1;
    border-top-right-radius: 10px;
    border-bottom-right-radius: 10px;
    margin: 5px 2px;
  }

  .border-b-dotted {
    border-bottom: 0.5px solid rgb(227, 227, 227);
  }

  .dark .border-b-dotted {
    border-bottom: 0.5px solid #9c9c9c82;
  }

  .add-post:hover {
    border: 1px solid #00000082;
    opacity: 100%;
  }

  .dark .add-post:hover {
    border: 1px solid #ececec82;
    opacity: 100%;
  }
</style>
