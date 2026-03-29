<script setup lang="ts">
import CategoryColor from '@/components/CategoryColor.vue'
import ConfirmationModal from '@/components/ConfirmationModal.vue'
import ToolTip from '@/components/ToolTip.vue'
import { createFormState, randomColor } from '@/helper'
import {
  categoryManager,
  DEFAULT_CATEGORY,
  PERM_CATEGORIES,
  type Category,
} from '@/schemas/category'
import { taskManager } from '@/schemas/task'
import { PencilIcon } from '@heroicons/vue/24/outline'
import { TrashIcon } from '@heroicons/vue/24/solid'
import { computed, nextTick, ref, type Ref } from 'vue'
import BaseView from './BaseView.vue'

const selectedCategory: Ref<Category | null> = ref(null)
const categories: Ref<Category[]> = ref(categoryManager.all())
const form = createFormState(
  {
    name: () => selectedCategory.value?.name ?? '',
    color: () => selectedCategory.value?.color ?? randomColor(),
  },
  {
    name: (x) => {
      const name = x.trim()
      if (name.length === 0) {
        return 'Name cannot be empty'
      }

      if (categories.value.some((c) => c.id !== selectedCategory.value?.id && c.name === name)) {
        return 'Name is already used for another category name'
      }
      return ''
    },
  },
)

function deleteCategory() {
  if (selectedCategory.value === null) {
    return
  }

  categoryManager.removeBy('id', selectedCategory.value.id)
  categories.value = categoryManager.all()
  if (categories.value.length === PERM_CATEGORIES.length) {
  }
}

async function updateCategory() {
  if (selectedCategory.value === null) {
    return
  }

  // Check for errors
  form.touchAll()
  await nextTick()

  if (form.state.hasErrors) {
    ;(document.querySelector('.input-error')! as HTMLInputElement).focus()
    return
  }

  categoryManager.updateBy('id', selectedCategory.value.id, {
    name: form.values.name,
    color: form.values.color,
  })
  categories.value = categoryManager.all()
}

const selectedCategoryTasks = computed(() => {
  return taskManager.filterBy('category', selectedCategory.value?.id ?? DEFAULT_CATEGORY)
})

const defaultCategory = computed(() => categoryManager.findBy('id', DEFAULT_CATEGORY))

const deleteModalRef: Ref<InstanceType<typeof ConfirmationModal> | null> = ref(null)
const updateModalRef: Ref<InstanceType<typeof ConfirmationModal> | null> = ref(null)

const canConfirm = computed(() => !form.state.hasErrors)
</script>

<template>
  <BaseView title="Manage Categories">
    <div
      v-if="categories.length > PERM_CATEGORIES.length"
      class="flex flex-col gap-2 w-1/2 mx-auto"
    >
      <div v-for="c in categories" :key="c.id">
        <div v-if="!PERM_CATEGORIES.some((x) => x === c.id)" class="flex justify-between">
          <div class="flex gap-2 align-middle">
            <CategoryColor :category="c" />
            <div>
              {{ c.name }}
            </div>
            <ToolTip tip="Update">
              <button
                class="btn btn-primary btn-xs"
                @click="
                  () => {
                    selectedCategory = c
                    form.reset()
                    updateModalRef?.showModal()
                  }
                "
              >
                <PencilIcon class="size-4" />
              </button>
            </ToolTip>
          </div>
          <ToolTip tip="Delete">
            <button
              class="btn btn-error btn-xs"
              @click="
                () => {
                  selectedCategory = c
                  deleteModalRef?.showModal()
                }
              "
            >
              <TrashIcon class="size-4" />
            </button>
          </ToolTip>
        </div>
      </div>
    </div>
    <div v-else>
      There are no custom categories, you can add one when either
      <RouterLink to="/create" class="link link-primary">creating a task</RouterLink>
      or
      <RouterLink to="/edit" class="link link-primary">editing a task</RouterLink>.
    </div>

    <ConfirmationModal
      ref="updateModalRef"
      title="Update Category"
      @confirm="updateCategory"
      :should-close="canConfirm"
      :positive="true"
    >
      <div class="flex flex-col w-full">
        <label
          class="w-full input"
          :class="{
            'input-error': form.touched.name && form.errors.name,
          }"
        >
          <span class="label">Name</span>
          <div class="flex justify-center">
            <input
              :placeholder="selectedCategory?.name"
              v-model="form.values.name"
              @blur="form.touch('name')"
            />
          </div>
        </label>
        <label v-if="form.errors.name && form.touched.name" class="label">
          <div class="label-text-alt text-error text-wrap">
            {{ form.errors.name }}
          </div>
        </label>
      </div>
      <div class="pt-2 flex gap-3">
        <div>Click the color to change it:</div>
        <div>
          <ToolTip tip="Change Color">
            <label class="cursor-pointer">
              <input type="color" v-model="form.values.color" class="sr-only" />
              <CategoryColor :color="form.values.color" :size="6" />
            </label>
          </ToolTip>
        </div>
      </div>

      <div class="flex justify-center py-3">
        <div v-if="form.state.hasErrors" class="alert alert-error min-w-max">
          <span>Please fix the errors above</span>
        </div>
      </div>
      <template #confirm> Update </template>
    </ConfirmationModal>

    <!-- Delete Category -->
    <ConfirmationModal ref="deleteModalRef" title="Delete Category" @confirm="deleteCategory">
      <div>Are you sure you want to delete category?</div>
      <div class="pt-4 text-center font-bold">"{{ selectedCategory?.name }}"</div>
      <div class="pt-6">
        It is currently used by
        <span class="font-bold">{{ selectedCategoryTasks.length }}</span>
        {{ selectedCategoryTasks.length == 1 ? 'task' : 'tasks' }}. Doing so will make them go back
        to the <CategoryColor :category="defaultCategory" :size="4" /> default category.
      </div>

      <template #confirm> Delete </template>
    </ConfirmationModal>
  </BaseView>
</template>
