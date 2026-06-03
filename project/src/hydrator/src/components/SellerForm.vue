<script setup lang="ts">
import { ref, computed } from 'vue';

const props = defineProps<{
  allData: Record<string, any[]>;
  tabs: string[];
  sellerCreatedTabs: Set<string>;
}>();

const emit = defineEmits<{
  (e: 'add-item',        payload: { category: string; item: any }): void;
  (e: 'delete-item',     payload: { category: string; id: string }): void;
  (e: 'update-item',     payload: { category: string; item: any }): void;
  (e: 'rename-category', payload: { oldName: string; newName: string }): void;
}>();

// ── Panel tabs ──────────────────────────────────────────────
type Panel = 'add' | 'manage';
const panel = ref<Panel>('add');

// ── ADD form ─────────────────────────────────────────────────
const form = ref({ categoryName: '', title: '', imageUrl: '', description: '' });
const uploadedFile = ref<File | null>(null);
const previewSrc   = ref('');
const errors       = ref<Record<string, string>>({});

const resolvedPreview = computed(() => previewSrc.value || form.value.imageUrl || '');

function onFileChange(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0];
  if (!file) return;
  uploadedFile.value = file;
  previewSrc.value   = URL.createObjectURL(file);
  form.value.imageUrl = '';
}
function onUrlInput() {
  uploadedFile.value = null;
  previewSrc.value   = '';
}

function validate() {
  errors.value = {};
  if (!form.value.categoryName.trim()) errors.value.categoryName = 'Category Name is required.';
  if (!form.value.title.trim())        errors.value.title        = 'Title is required.';
  if (!resolvedPreview.value)          errors.value.image        = 'Provide an image file or URL.';
  return Object.keys(errors.value).length === 0;
}

function submitAdd() {
  if (!validate()) return;
  const category = form.value.categoryName.trim();
  const exists   = props.tabs.includes(category);

  const newItem = {
    id:          `seller_${Date.now()}`,
    title:       form.value.title.trim(),
    image:       resolvedPreview.value,
    description: form.value.description.trim(),
  };

  console.log(exists
    ? `[Seller Form] Category found: "${category}"`
    : `[Seller Form] Category created: "${category}"`);

  emit('add-item', { category, item: newItem });
  console.log('[Seller Form] Item added:', newItem);
  console.log('[Seller Form] Current category data:', props.allData);

  form.value     = { categoryName: '', title: '', imageUrl: '', description: '' };
  uploadedFile.value = null;
  previewSrc.value   = '';
  errors.value       = {};
}

// ── MANAGE ────────────────────────────────────────────────────
const manageCategory = ref('');
const manageCategoryItems = computed(() =>
  manageCategory.value ? (props.allData[manageCategory.value] ?? []) : []
);

// Rename state
const isRenaming   = ref(false);
const renameValue  = ref('');

function startRename() {
  renameValue.value = manageCategory.value;
  isRenaming.value  = true;
}
function saveRename() {
  const newName = renameValue.value.trim();
  if (!newName || newName === manageCategory.value) { isRenaming.value = false; return; }
  emit('rename-category', { oldName: manageCategory.value, newName });
  manageCategory.value = newName;
  isRenaming.value = false;
}
function cancelRename() {
  isRenaming.value = false;
}

// Switch to Add panel with category pre-filled
function addItemToCategory() {
  form.value = { categoryName: manageCategory.value, title: '', imageUrl: '', description: '' };
  uploadedFile.value = null;
  previewSrc.value = '';
  errors.value = {};
  panel.value = 'add';
}

// Edit state
const editingId  = ref<string | null>(null);
const editTitle  = ref('');
const editDesc   = ref('');
const editImgUrl = ref('');
const editPreview = ref('');

const editResolvedPreview = computed(() => editPreview.value || editImgUrl.value || '');

function startEdit(item: any) {
  editingId.value  = item.id;
  editTitle.value  = item.title;
  editDesc.value   = item.description ?? '';
  editImgUrl.value = item.image ?? '';
  editPreview.value = '';
}

function onEditFileChange(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0];
  if (!file) return;
  editPreview.value = URL.createObjectURL(file);
  editImgUrl.value  = '';
}
function onEditUrlInput() {
  editPreview.value = '';
}

function saveEdit() {
  if (!editingId.value || !manageCategory.value) return;
  const updated = {
    id:          editingId.value,
    title:       editTitle.value.trim(),
    image:       editResolvedPreview.value,
    description: editDesc.value.trim(),
  };
  emit('update-item', { category: manageCategory.value, item: updated });
  console.log('[Seller Form] Item updated:', updated);
  cancelEdit();
}

function cancelEdit() {
  editingId.value = null;
  editTitle.value = editDesc.value = editImgUrl.value = editPreview.value = '';
}

function deleteItem(id: string) {
  if (!manageCategory.value) return;
  emit('delete-item', { category: manageCategory.value, id });
  console.log(`[Seller Form] Item deleted: id=${id} from "${manageCategory.value}"`);
  if (editingId.value === id) cancelEdit();
}
</script>

<template>
  <div class="w-full bg-white dark:bg-gray-900 rounded-2xl shadow-lg overflow-hidden">

    <!-- Panel switcher -->
    <div class="flex border-b border-gray-200 dark:border-gray-700">
      <button
        v-for="p in (['add', 'manage'] as const)" :key="p"
        @click="panel = p"
        class="flex-1 py-3 text-sm font-bold tracking-wide transition"
        :class="panel === p
          ? 'bg-black dark:bg-white text-white dark:text-black'
          : 'text-gray-400 hover:text-gray-700 dark:hover:text-gray-200'"
      >
        {{ p === 'add' ? '＋ Add Item' : '✎ Manage Items' }}
      </button>
    </div>

    <!-- ── ADD PANEL ── -->
    <div v-if="panel === 'add'" class="p-5 md:p-8">
      <div class="grid grid-cols-1 md:grid-cols-2 gap-5">

        <!-- Category Name -->
        <div class="flex flex-col gap-1">
          <label class="text-sm font-semibold text-gray-600 dark:text-gray-300">
            Category Name <span class="text-red-500">*</span>
          </label>
          <input v-model="form.categoryName" list="cat-list"
            placeholder="e.g. Men, Women, Sports…"
            class="border border-gray-300 dark:border-gray-600 rounded-xl px-4 py-2.5 text-sm bg-white dark:bg-gray-800 dark:text-white outline-none focus:ring-2 focus:ring-black dark:focus:ring-white transition"
          />
          <datalist id="cat-list">
            <option v-for="tab in tabs" :key="tab" :value="tab" />
          </datalist>
          <span v-if="errors.categoryName" class="text-xs text-red-500">{{ errors.categoryName }}</span>
        </div>

        <!-- Title -->
        <div class="flex flex-col gap-1">
          <label class="text-sm font-semibold text-gray-600 dark:text-gray-300">
            Title <span class="text-red-500">*</span>
          </label>
          <input v-model="form.title" placeholder="e.g. T-Shirts, Sneakers…"
            class="border border-gray-300 dark:border-gray-600 rounded-xl px-4 py-2.5 text-sm bg-white dark:bg-gray-800 dark:text-white outline-none focus:ring-2 focus:ring-black dark:focus:ring-white transition"
          />
          <span v-if="errors.title" class="text-xs text-red-500">{{ errors.title }}</span>
        </div>

        <!-- Image Upload -->
        <div class="flex flex-col gap-1">
          <label class="text-sm font-semibold text-gray-600 dark:text-gray-300">Image Upload</label>
          <input type="file" accept="image/*" @change="onFileChange"
            class="text-sm text-gray-500 dark:text-gray-400 file:mr-3 file:py-2 file:px-4 file:rounded-full file:border-0 file:text-xs file:font-semibold file:bg-black file:text-white dark:file:bg-white dark:file:text-black cursor-pointer"
          />
        </div>

        <!-- Image URL -->
        <div class="flex flex-col gap-1">
          <label class="text-sm font-semibold text-gray-600 dark:text-gray-300">Image URL</label>
          <input v-model="form.imageUrl" @input="onUrlInput" placeholder="https://…"
            class="border border-gray-300 dark:border-gray-600 rounded-xl px-4 py-2.5 text-sm bg-white dark:bg-gray-800 dark:text-white outline-none focus:ring-2 focus:ring-black dark:focus:ring-white transition"
          />
          <span v-if="errors.image" class="text-xs text-red-500">{{ errors.image }}</span>
        </div>

        <!-- Description -->
        <div class="flex flex-col gap-1 md:col-span-2">
          <label class="text-sm font-semibold text-gray-600 dark:text-gray-300">Description (optional)</label>
          <textarea v-model="form.description" rows="2" placeholder="Short description…"
            class="border border-gray-300 dark:border-gray-600 rounded-xl px-4 py-2.5 text-sm bg-white dark:bg-gray-800 dark:text-white outline-none focus:ring-2 focus:ring-black dark:focus:ring-white transition resize-none"
          />
        </div>
      </div>

      <!-- Image preview -->
      <div v-if="resolvedPreview" class="mt-5">
        <p class="text-xs font-semibold text-gray-500 dark:text-gray-400 mb-2">Preview</p>
        <img :src="resolvedPreview" alt="preview"
          class="h-48 w-full object-cover rounded-2xl border border-gray-200 dark:border-gray-700"
        />
      </div>

      <button @click="submitAdd"
        class="mt-6 w-full py-3 rounded-full bg-black dark:bg-white text-white dark:text-black text-sm font-bold tracking-wider hover:opacity-80 transition"
      >
        Add to Collage
      </button>
    </div>

    <!-- ── MANAGE PANEL ── -->
    <div v-else class="p-5 md:p-8">

      <!-- Category selector -->
      <div class="flex flex-col gap-1 mb-4">
        <label class="text-sm font-semibold text-gray-600 dark:text-gray-300">Select Category</label>
        <select v-model="manageCategory" @change="cancelEdit(); isRenaming = false"
          class="border border-gray-300 dark:border-gray-600 rounded-xl px-4 py-2.5 text-sm bg-white dark:bg-gray-800 dark:text-white outline-none focus:ring-2 focus:ring-black dark:focus:ring-white transition"
        >
          <option value="" disabled>Choose a category…</option>
          <option v-for="tab in tabs" :key="tab" :value="tab">{{ tab }}</option>
        </select>
      </div>

      <!-- Rename category -->
      <div v-if="manageCategory" class="mb-5">
        <div v-if="!isRenaming" class="flex items-center gap-2">
          <span class="text-xs text-gray-400 dark:text-gray-500">Category:</span>
          <span class="text-sm font-bold text-gray-800 dark:text-white">{{ manageCategory }}</span>
          <button @click="startRename"
            class="ml-auto px-3 py-1 text-xs font-semibold rounded-full border border-gray-300 dark:border-gray-600 hover:bg-gray-100 dark:hover:bg-gray-700 transition"
          >✎ Rename</button>
        </div>
        <div v-else class="flex items-center gap-2">
          <input v-model="renameValue" @keyup.enter="saveRename" @keyup.escape="cancelRename"
            class="flex-1 border border-gray-300 dark:border-gray-600 rounded-xl px-3 py-2 text-sm bg-white dark:bg-gray-800 dark:text-white outline-none focus:ring-2 focus:ring-black dark:focus:ring-white transition"
          />
          <button @click="saveRename"
            class="px-3 py-1.5 text-xs font-bold rounded-full bg-black dark:bg-white text-white dark:text-black hover:opacity-80 transition"
          >Save</button>
          <button @click="cancelRename"
            class="px-3 py-1.5 text-xs font-bold rounded-full border border-gray-300 dark:border-gray-600 hover:bg-gray-100 dark:hover:bg-gray-700 transition"
          >Cancel</button>
        </div>
      </div>

      <!-- Empty state -->
      <div v-if="manageCategory && manageCategoryItems.length === 0" class="text-center py-8">
        <p class="text-sm text-gray-400 mb-4">No items in "{{ manageCategory }}" yet.</p>
        <button @click="addItemToCategory"
          class="px-5 py-2.5 rounded-full bg-black dark:bg-white text-white dark:text-black text-xs font-bold hover:opacity-80 transition"
        >＋ Add first item here</button>
      </div>

      <!-- Add more button when items exist -->
      <div v-if="manageCategory && manageCategoryItems.length > 0" class="mb-4">
        <button @click="addItemToCategory"
          class="w-full py-2.5 rounded-xl border-2 border-dashed border-gray-300 dark:border-gray-600 text-sm font-semibold text-gray-400 hover:border-black hover:text-black dark:hover:border-white dark:hover:text-white transition"
        >＋ Add another item to "{{ manageCategory }}"</button>
      </div>

      <!-- Items list -->
      <TransitionGroup name="fade-list" tag="div" class="flex flex-col gap-3">
        <div
          v-for="item in manageCategoryItems" :key="item.id"
          class="border border-gray-200 dark:border-gray-700 rounded-2xl overflow-hidden"
        >
          <!-- Collapsed row -->
          <div v-if="editingId !== item.id"
            class="flex items-center gap-3 p-3"
          >
            <img :src="item.image" alt=""
              class="w-16 h-16 object-cover rounded-xl flex-shrink-0 bg-gray-100 dark:bg-gray-800"
            />
            <div class="flex-1 min-w-0">
              <p class="font-bold text-sm text-gray-800 dark:text-white truncate">{{ item.title }}</p>
              <p v-if="item.description" class="text-xs text-gray-400 truncate">{{ item.description }}</p>
            </div>
            <div class="flex gap-2 flex-shrink-0">
              <button @click="startEdit(item)"
                class="px-3 py-1.5 text-xs font-semibold rounded-full border border-gray-300 dark:border-gray-600 hover:bg-gray-100 dark:hover:bg-gray-700 transition"
              >Edit</button>
              <button @click="deleteItem(item.id)"
                class="px-3 py-1.5 text-xs font-semibold rounded-full bg-red-500 text-white hover:bg-red-600 transition"
              >Delete</button>
            </div>
          </div>

          <!-- Edit inline form -->
          <div v-else class="p-4 flex flex-col gap-3 bg-gray-50 dark:bg-gray-800">
            <p class="text-xs font-bold text-gray-500 dark:text-gray-400 uppercase tracking-widest mb-1">Editing item</p>

            <div class="grid grid-cols-1 md:grid-cols-2 gap-3">
              <!-- Title -->
              <div class="flex flex-col gap-1">
                <label class="text-xs font-semibold text-gray-500 dark:text-gray-400">Title</label>
                <input v-model="editTitle"
                  class="border border-gray-300 dark:border-gray-600 rounded-xl px-3 py-2 text-sm bg-white dark:bg-gray-900 dark:text-white outline-none focus:ring-2 focus:ring-black dark:focus:ring-white transition"
                />
              </div>
              <!-- Image URL -->
              <div class="flex flex-col gap-1">
                <label class="text-xs font-semibold text-gray-500 dark:text-gray-400">Image URL</label>
                <input v-model="editImgUrl" @input="onEditUrlInput" placeholder="https://…"
                  class="border border-gray-300 dark:border-gray-600 rounded-xl px-3 py-2 text-sm bg-white dark:bg-gray-900 dark:text-white outline-none focus:ring-2 focus:ring-black dark:focus:ring-white transition"
                />
              </div>
              <!-- File upload -->
              <div class="flex flex-col gap-1 md:col-span-2">
                <label class="text-xs font-semibold text-gray-500 dark:text-gray-400">Replace Image (upload)</label>
                <input type="file" accept="image/*" @change="onEditFileChange"
                  class="text-sm text-gray-500 file:mr-3 file:py-1.5 file:px-3 file:rounded-full file:border-0 file:text-xs file:font-semibold file:bg-black file:text-white cursor-pointer"
                />
              </div>
              <!-- Description -->
              <div class="flex flex-col gap-1 md:col-span-2">
                <label class="text-xs font-semibold text-gray-500 dark:text-gray-400">Description</label>
                <textarea v-model="editDesc" rows="2"
                  class="border border-gray-300 dark:border-gray-600 rounded-xl px-3 py-2 text-sm bg-white dark:bg-gray-900 dark:text-white outline-none focus:ring-2 focus:ring-black dark:focus:ring-white transition resize-none"
                />
              </div>
            </div>

            <!-- Edit preview -->
            <div v-if="editResolvedPreview" class="mt-1">
              <p class="text-xs font-semibold text-gray-400 mb-1">Preview</p>
              <img :src="editResolvedPreview" alt="edit-preview"
                class="h-36 w-full object-cover rounded-xl border border-gray-200 dark:border-gray-700"
              />
            </div>

            <div class="flex gap-2 mt-1">
              <button @click="saveEdit"
                class="flex-1 py-2 rounded-full bg-black dark:bg-white text-white dark:text-black text-xs font-bold hover:opacity-80 transition"
              >Save Changes</button>
              <button @click="cancelEdit"
                class="flex-1 py-2 rounded-full border border-gray-300 dark:border-gray-600 text-xs font-bold hover:bg-gray-100 dark:hover:bg-gray-700 transition"
              >Cancel</button>
            </div>
          </div>
        </div>
      </TransitionGroup>
    </div>
  </div>
</template>

<style scoped>
.fade-list-move,
.fade-list-enter-active,
.fade-list-leave-active { transition: all 0.3s ease; }
.fade-list-enter-from,
.fade-list-leave-to { opacity: 0; transform: translateY(8px); }
.fade-list-leave-active { position: absolute; width: 100%; }
</style>
