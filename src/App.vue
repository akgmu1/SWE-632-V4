<script setup lang="ts">
import {
  ArchiveBoxIcon,
  BugAntIcon,
  HomeIcon,
  PencilSquareIcon,
  PlusIcon,
  PresentationChartLineIcon,
  TrashIcon,
} from '@heroicons/vue/24/outline'
import { RouterLink, RouterView } from 'vue-router'
import { notifications } from './notification'

const IS_DEV = import.meta.env.DEV
</script>

<template>
  <div class="drawer drawer-open">
    <input id="my-drawer-1" type="checkbox" class="drawer-toggle" />
    <div class="drawer-content">
      <!-- The primary content here -->
      <div class="px-4">
        <RouterView />
      </div>
    </div>
    <div class="drawer-side">
      <label for="my-drawer-1" aria-label="close sidebar" class="drawer-overlay"></label>
      <div class="bg-base-300 min-h-full">
        <RouterLink to="/">
          <div class="flex pt-2 justify-center items-center gap-2">
            <img src="/logo.png" class="size-9 rounded" />
            <div class="text-2xl font-semibold">Task Manager</div>
          </div>
        </RouterLink>
        <ul class="menu w-64 p-4 gap-1">
          <!-- Sidebar content here -->
          <li>
            <RouterLink to="/" active-class="bg-primary text-primary-content">
              <HomeIcon class="size-5" />
              Home
            </RouterLink>
          </li>
          <li>
            <RouterLink to="/create" active-class="bg-primary text-primary-content">
              <PlusIcon class="size-5" />
              Create Task
            </RouterLink>
          </li>
          <li>
            <RouterLink to="/edit" active-class="bg-primary text-primary-content">
              <PencilSquareIcon class="size-5" />
              Edit Task
            </RouterLink>
          </li>
          <li>
            <RouterLink to="/delete" active-class="bg-primary text-primary-content">
              <TrashIcon class="size-5" />
              Delete Task
            </RouterLink>
          </li>
          <li>
            <RouterLink to="/categories" active-class="bg-primary text-primary-content">
              <ArchiveBoxIcon class="size-5" />
              Manage Categories
            </RouterLink>
          </li>
          <li>
            <RouterLink to="/stats" active-class="bg-primary text-primary-content">
              <PresentationChartLineIcon class="size-5" />
              Statistics
            </RouterLink>
          </li>
          <li v-if="IS_DEV">
            <RouterLink to="/debug" active-class="bg-primary text-primary-content">
              <BugAntIcon class="size-5" />
              Debug
            </RouterLink>
          </li>
        </ul>
      </div>
    </div>
  </div>
  <div class="toast toast-end toast-bottom">
    <div
      v-for="n in notifications"
      class="alert"
      :class="{
        'alert-success': n.type === 'success',
        'alert-info': n.type === 'info',
        'alert-warning': n.type === 'warning',
        'alert-error': n.type === 'error',
      }"
    >
      {{ n.message }}
    </div>
  </div>
</template>
