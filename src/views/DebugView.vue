<script setup lang="ts">
import { dateTrim, getRandomElement, randomDateNear, randomNumberNear } from '@/helper'
import { notify } from '@/notification'
import { DEFAULT_CATEGORY } from '@/schemas/category'
import { taskManager, type CreateTask } from '@/schemas/task'
import { timeEntryManager } from '@/schemas/timeEntry'
import BaseView from './BaseView.vue'

function resetTasks() {
  taskManager.reset(true)
  notify('Cleared tasks', 'info')
}

function addTestTasks() {
  const TASKS: CreateTask[] = [
    {
      title: 'Test 1',
      completed: false,
      category: DEFAULT_CATEGORY,
      dueDate: dateTrim(new Date()),
      created: new Date(),
    },
    {
      title: 'Test 2',
      completed: true,
      category: DEFAULT_CATEGORY,
      dueDate: dateTrim(new Date()),
      created: new Date(),
    },
    {
      title: 'Test 3',
      completed: false,
      category: DEFAULT_CATEGORY,
      dueDate: dateTrim(new Date()),
      created: new Date(),
    },
  ]
  for (const task of TASKS) {
    taskManager.add(task)
  }
  notify('Added test task', 'info')
}

function logRandom() {
  const tasks = taskManager.all()
  const task = getRandomElement(tasks)
  if (!task) return
  timeEntryManager.add({
    taskId: task.id,
    minutes: randomNumberNear(60, 5, 10, 210),
    date: randomDateNear(new Date()),
    note: '',
  })
  notify('Logged random entry', 'info')
}
</script>

<template>
  <BaseView title="Debug">
    <div class="flex flex-col gap-2">
      <div>
        <RouterLink to="/" class="btn btn-primary">Go Back Home</RouterLink>
      </div>
      <div>
        <button @click="resetTasks" class="btn btn-primary">Reset tasks</button>
      </div>
      <div>
        <button @click="addTestTasks" class="btn btn-primary">Add test tasks</button>
      </div>
      <div>
        <button
          @click="
            () => {
              notify('test', 'error')
            }
          "
          class="btn btn-primary"
        >
          Test notification
        </button>
      </div>
      <div>
        <button @click="logRandom" class="btn btn-primary">Add Random Log Entry</button>
      </div>
    </div>
  </BaseView>
  <div id="my-toast" class="toast toast-center toast-middle hidden">
    <div class="alert alert-info">
      <span>Done</span>
    </div>
  </div>
</template>
