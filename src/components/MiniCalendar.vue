<script setup lang="ts">
import { isSameDay } from '@/helper'
import { ArrowLeftIcon, ArrowRightIcon } from '@heroicons/vue/24/solid'
import { computed, ref, watch } from 'vue'

interface Props {
  modelValue?: Date
  month?: Date
}

const props = defineProps<Props>()

interface Emits {
  (e: 'update:modelValue', value: Date): void
}

const emit = defineEmits<Emits>()

const today = new Date()

const internalMonth = ref(
  new Date(
    (props.month ?? props.modelValue ?? today).getFullYear(),
    (props.month ?? props.modelValue ?? today).getMonth(),
    1,
  ),
)

watch(
  () => props.month,
  (m) => {
    if (m) internalMonth.value = new Date(m.getFullYear(), m.getMonth(), 1)
  },
)

function prevMonth() {
  internalMonth.value = new Date(
    internalMonth.value.getFullYear(),
    internalMonth.value.getMonth() - 1,
    1,
  )
}

function nextMonth() {
  internalMonth.value = new Date(
    internalMonth.value.getFullYear(),
    internalMonth.value.getMonth() + 1,
    1,
  )
}

const days = computed(() => {
  // Get the first and last day of the month
  const first = new Date(internalMonth.value.getFullYear(), internalMonth.value.getMonth(), 1)
  const last = new Date(internalMonth.value.getFullYear(), internalMonth.value.getMonth() + 1, 0)

  const start = new Date(first)
  start.setDate(first.getDate() - first.getDay())

  const end = new Date(last)
  end.setDate(last.getDate() + (6 - last.getDay()))

  // Create the list of days
  const result: Date[] = []
  const cursor = new Date(start)

  while (cursor <= end) {
    result.push(new Date(cursor))
    cursor.setDate(cursor.getDate() + 1)
  }

  // Ensure it is a week long
  while (result.length % 7 !== 0) {
    result.pop()
  }

  // Remove the last week if it is not part of this month
  const lastWeek = result.slice(-7)
  if (lastWeek.every((d) => d.getMonth() !== internalMonth.value.getMonth())) {
    result.splice(-7)
  }

  return result
})

const monthLabel = computed(() =>
  internalMonth.value.toLocaleDateString(undefined, {
    month: 'long',
    year: 'numeric',
  }),
)

function onSelect(date: Date) {
  emit('update:modelValue', date)

  if (date.getMonth() !== internalMonth.value.getMonth()) {
    internalMonth.value = new Date(date.getFullYear(), date.getMonth(), 1)
  }
}
</script>

<template>
  <div class="w-72 bg-base-100 text-base-content rounded-box shadow p-3">
    <!-- Header -->
    <div class="flex items-center justify-between mb-2">
      <button class="btn btn-ghost btn-xs" @click="prevMonth">
        <ArrowLeftIcon class="size-4" />
      </button>

      <div class="font-medium text-sm">
        {{ monthLabel }}
      </div>

      <button class="btn btn-ghost btn-xs" @click="nextMonth">
        <ArrowRightIcon class="size-4" />
      </button>
    </div>

    <!-- Days of the Week -->
    <div class="grid grid-cols-7 text-[10px] opacity-60 mb-1 text-center">
      <div v-for="d in ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat']" :key="d">
        {{ d }}
      </div>
    </div>

    <!-- Grid of Days -->
    <div class="grid grid-cols-7 gap-1">
      <button
        v-for="date in days"
        :key="date.toISOString()"
        @click="onSelect(date)"
        class="aspect-square rounded-lg p-1 text-left transition-colors text-xs flex flex-col"
        :class="[
          date.getMonth() !== internalMonth.getMonth() ? 'opacity-30' : '',
          isSameDay(date, modelValue) ? 'bg-primary text-primary-content' : 'hover:bg-base-200',
        ]"
      >
        <slot
          name="day"
          :date="date"
          :isSelected="isSameDay(date, modelValue)"
          :isToday="isSameDay(date, today)"
          :isCurrentMonth="date.getMonth() === internalMonth.getMonth()"
        >
          <span
            class="text-[11px]"
            :class="
              isSameDay(date, today) && !isSameDay(date, modelValue)
                ? 'text-primary font-semibold'
                : ''
            "
          >
            {{ date.getDate() }}
          </span>
        </slot>
      </button>
    </div>
  </div>
</template>
