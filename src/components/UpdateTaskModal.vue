<script setup lang="ts">
import { ToolTipDirection } from '@/enums'
import { createFormState, dateToYYYYMMDD, dateTrim, randomColor, triggerAddClass } from '@/helper'
import {
  categoryManager,
  DEFAULT_CATEGORY,
  META_ADD_NEW_CATEGORY,
  type Category,
} from '@/schemas/category'
import { subtaskManager, type Subtask } from '@/schemas/subtask'
import type { Task } from '@/schemas/task'
import { computed, nextTick, onBeforeUnmount, ref, watch, type Ref } from 'vue'
import BaseModal from './BaseModal.vue'
import CategoryColor from './CategoryColor.vue'
import ConfirmationModal from './ConfirmationModal.vue'
import ToolTip from './ToolTip.vue'

interface Emits {
  (e: 'updateTask', task: Task, subtask: boolean): void
}

const emits = defineEmits<Emits>()

const subtaskForm = createFormState(
  {
    newSubtaskText: '',
  },
  {
    newSubtaskText: (x) => {
      if (x.trim().length === 0) {
        return 'Subtask text cannot be empty'
      }

      return ''
    },
  },
)

let timeout: number | undefined = undefined
watch(subtaskForm.state, (s) => {
  if (s.hasErrors) {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      subtaskForm.clearTouchAndErrors()
    }, 3000)
  }
})

onBeforeUnmount(() => {
  clearTimeout(timeout)
})

const task: Ref<Task | undefined> = ref(undefined)
const subtasks: Ref<Subtask[]> = ref([])
const tempSubtasks = ref<string[]>([])

const categories: Ref<Category[]> = ref(categoryManager.all())
const existingCategory: Ref<number | undefined> = ref()

const form = createFormState(
  {
    title: () => task.value?.title ?? '',
    selectedCategory: () => task.value?.category ?? DEFAULT_CATEGORY,
    newCategoryName: '',
    newCategoryColor: randomColor,
    dueDate: () => task.value?.dueDate ?? new Date(),
  },
  {
    title: (x) => (x.trim().length > 0 ? '' : 'Title cannot be empty'),
    newCategoryName: (x, form) => {
      // No errors when not adding one
      existingCategory.value = undefined
      if (form.selectedCategory !== META_ADD_NEW_CATEGORY) {
        return ''
      }

      const name = x.trim()
      if (name.length === 0) {
        return 'Category name cannot be empty'
      }

      for (const c of categories.value) {
        if (c.name === name) {
          existingCategory.value = c.id
          return 'Category name already taken, please pick another name or select that category'
        }
      }

      return ''
    },
  },
)

const currentCategory = computed(() => {
  return categoryManager.findBy('id', form.values.selectedCategory)!
})

const isAddingNewCategory = computed(() => form.values.selectedCategory === META_ADD_NEW_CATEGORY)

function onCategoryChange(val: number) {
  form.values.selectedCategory = val
  if (val !== META_ADD_NEW_CATEGORY) {
    form.values.newCategoryName = ''
    form.values.newCategoryColor = randomColor()
  }
}

const addSubtaskRef: Ref<HTMLElement | null> = ref(null)
const subtaskTextRef: Ref<HTMLInputElement | null> = ref(null)
async function addSubtask() {
  subtaskForm.touchAll()
  await nextTick()

  if (subtaskForm.state.hasErrors) {
    triggerAddClass(addSubtaskRef.value!, 'animate-shake')
    await nextTick()
    subtaskTextRef.value!.focus()
    return
  }

  const text = subtaskForm.values.newSubtaskText.trim()
  tempSubtasks.value.push(text)
  subtaskForm.reset()
}

function removeTempSubtask(index: number) {
  tempSubtasks.value.splice(index, 1)
}

function toggleSubtask(subtask: Subtask, completed: boolean) {
  if (!task.value) return

  subtaskManager.updateBy('id', subtask.id, {
    completed,
  })

  emits('updateTask', task.value, true)
}

function removeSubtask(subtask: Subtask) {
  if (!task.value) return

  subtaskManager.removeBy('id', subtask.id)
  subtasks.value = subtaskManager.filterBy('taskId', task.value.id)

  emits('updateTask', task.value, true)
}

const modalRef: Ref<InstanceType<typeof BaseModal> | null> = ref(null)

defineExpose({
  showModal: (t: Task) => {
    categories.value = categoryManager.all()
    task.value = t

    subtasks.value = subtaskManager.filterBy('taskId', t.id)
    tempSubtasks.value = []

    subtaskForm.reset()
    form.reset()

    modalRef.value!.showModal()
  },
  close: () => {
    modalRef.value!.close()
  },
})

const canConfirm = computed(() => {
  if (subtaskForm.state.hasErrors) return false
  if (form.state.hasErrors) return false
  return true
})

async function onConfirm() {
  form.touchAll()
  await nextTick()

  if (form.state.hasErrors) {
    await nextTick()
    ;(document.querySelector('.input-error')! as HTMLInputElement).focus()
    return
  }

  if (!canConfirm.value || !task.value) return

  let finalCategory: number = DEFAULT_CATEGORY

  if (isAddingNewCategory.value) {
    const newName = form.values.newCategoryName.trim()
    finalCategory = categoryManager.add({
      name: newName,
      color: form.values.newCategoryColor,
    })
    form.values.selectedCategory = finalCategory
  } else {
    finalCategory = form.values.selectedCategory
  }

  const updatedTask: Task = {
    ...task.value,
    title: form.values.title.trim(),
    category: finalCategory,
    dueDate: dateTrim(form.values.dueDate),
  }

  emits('updateTask', updatedTask, true)

  // Existing task flow: attach newly added subtasks immediately.
  // If your create page uses a different component, copy this same pattern there.
  tempSubtasks.value.forEach((text) => {
    subtaskManager.add({
      taskId: updatedTask.id,
      completed: false,
      text,
    })
  })

  subtasks.value = subtaskManager.filterBy('taskId', updatedTask.id)
  tempSubtasks.value = []

  task.value = undefined
  subtaskForm.reset()
  form.reset()
}
</script>

<template>
  <ConfirmationModal
    ref="modalRef"
    title="Update Task"
    @confirm="onConfirm"
    :should-close="canConfirm"
    :positive="true"
  >
    <div class="container mx-auto pt-4 text-center">
      <div class="flex flex-col gap-4 sm:flex-row">
        <div class="flex flex-col w-full">
          <label
            class="w-full input"
            :class="{
              'input-error': form.touched.title && form.errors.title,
            }"
          >
            <span class="label">Title</span>
            <div class="flex justify-center">
              <input
                v-model="form.values.title"
                @blur="form.touch('title')"
                type="text"
                placeholder="Title"
                class="w-full"
                @keyup.enter="onConfirm"
              />
            </div>
          </label>

          <label v-if="form.errors.title && form.touched.title" class="label">
            <div class="label-text-alt text-error text-wrap">
              {{ form.errors.title }}
            </div>
          </label>
        </div>

        <input
          type="date"
          :value="dateToYYYYMMDD(form.values.dueDate)"
          @input="
            form.values.dueDate = dateTrim(
              ($event.target as HTMLInputElement).valueAsDate ?? new Date(),
              true,
            )
          "
          class="input-bordered input w-full"
        />
      </div>

      <div class="mx-auto mt-6 flex items-center gap-3">
        <CategoryColor :category="currentCategory" />

        <select
          class="select-bordered select w-full"
          :value="form.values.selectedCategory"
          @change="onCategoryChange(Number(($event.target as HTMLSelectElement).value))"
        >
          <option value="" disabled>Selected Category (optional)</option>
          <option v-for="c in categories" :key="c.id" :value="c.id">
            {{ c.id === META_ADD_NEW_CATEGORY ? '+ Add new category…' : c.name }}
          </option>
        </select>
      </div>

      <div v-if="isAddingNewCategory" class="mt-6 flex items-center gap-3">
        <ToolTip tip="Change Color" :direction="ToolTipDirection.Right">
          <button
            type="button"
            @click="
              () => {
                form.values.newCategoryColor = randomColor()
              }
            "
            class="cursor-pointer"
          >
            <CategoryColor :color="form.values.newCategoryColor" />
          </button>
        </ToolTip>
        <div class="flex flex-col w-full">
          <input
            v-model="form.values.newCategoryName"
            @blur="form.touch('newCategoryName')"
            type="text"
            placeholder="New category name"
            class="input-bordered input w-full"
            :class="{
              'input-error': form.touched.newCategoryName && form.errors.newCategoryName,
            }"
            @keyup.enter="onConfirm"
          />

          <label v-if="form.errors.newCategoryName && form.touched.newCategoryName" class="label">
            <div class="label-text-alt text-error text-wrap text-left">
              {{ form.errors.newCategoryName }}
            </div>

            <template v-if="existingCategory !== undefined">
              <button
                class="btn btn-secondary btn-sm"
                @click="
                  () => {
                    form.values.selectedCategory = existingCategory!
                    form.values.newCategoryName = ''
                    form.errors.newCategoryName = ''
                    existingCategory = undefined
                  }
                "
              >
                Click to select that category
              </button>
            </template>
          </label>
        </div>
      </div>

      <div class="mt-6 text-left">
        <div class="mb-2 font-semibold">Subtasks</div>

        <div v-if="subtasks.length || tempSubtasks.length" class="space-y-2">
          <div v-for="s in subtasks" :key="s.id" class="flex items-center gap-2">
            <input
              type="checkbox"
              class="checkbox"
              :checked="s.completed"
              @change="toggleSubtask(s, ($event.target as HTMLInputElement).checked)"
            />

            <div class="flex-1" :class="{ 'line-through opacity-60': s.completed }">
              {{ s.text }}
            </div>

            <button type="button" class="btn btn-ghost btn-xs" @click="removeSubtask(s)">
              Remove
            </button>
          </div>

          <div
            v-for="(s, index) in tempSubtasks"
            :key="`temp-${index}`"
            class="flex items-center gap-2"
          >
            <div class="flex-1">
              {{ s }}
            </div>

            <button type="button" class="btn btn-ghost btn-xs" @click="removeTempSubtask(index)">
              Remove
            </button>
          </div>
        </div>

        <div class="flex flex-col mt-5">
          <div class="flex gap-2">
            <input
              ref="subtaskTextRef"
              v-model="subtaskForm.values.newSubtaskText"
              class="input-bordered input w-full"
              placeholder="Add a subtask..."
              @keydown.enter.prevent="addSubtask"
              :class="{
                'input-error':
                  subtaskForm.errors.newSubtaskText && subtaskForm.touched.newSubtaskText,
              }"
            />
            <button ref="addSubtaskRef" type="button" class="btn btn-primary" @click="addSubtask">
              Add
            </button>
          </div>
          <label
            v-if="subtaskForm.errors.newSubtaskText && subtaskForm.touched.newSubtaskText"
            class="label"
          >
            <div class="label-text-alt text-error text-wrap">
              {{ subtaskForm.errors.newSubtaskText }}
            </div>
          </label>
        </div>
      </div>
    </div>

    <div class="flex justify-center mt-5">
      <div v-if="form.state.hasErrors" class="alert alert-error min-w-max">
        <span>Please fix the errors above</span>
      </div>
    </div>

    <template #confirm> Update </template>
  </ConfirmationModal>
</template>
