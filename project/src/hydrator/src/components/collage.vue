<script setup lang="ts">
import { ref, computed, watch, onMounted, nextTick } from 'vue';
import SellerForm from './SellerForm.vue';

// ── localStorage helpers ──────────────────────────────────────
const STORAGE_KEY = 'collage_seller_data';

function loadFromStorage(): { categories: Record<string, any[]>; sellerTabs: string[]; allTabs: string[] } {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (raw) {
      const parsed = JSON.parse(raw);
      // Migration: old data had no allTabs — discard it so defaults are used cleanly
      if (!parsed.allTabs || !parsed.allTabs.length) {
        localStorage.removeItem(STORAGE_KEY);
        return { categories: {}, sellerTabs: [], allTabs: [] };
      }
      return parsed;
    }
  } catch {}
  return { categories: {}, sellerTabs: [], allTabs: [] };
}

function saveToStorage(categories: Record<string, any[]>, sellerTabs: string[], allTabs: string[]) {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify({ categories, sellerTabs, allTabs }));
  } catch {}
}

const props = defineProps<{
  mode?: "sticky" | "normal"
}>();
const mode = computed(() => props.mode ?? "normal");

// ── Load storage first, everything depends on it ─────────────
const DEFAULT_TABS = ['Men', 'Women', 'Beauty', 'Kids', 'Home', 'himanshu', 'gooo'];
const DEFAULT_SELLER_TABS = ['himanshu', 'gooo'];
const _saved = loadFromStorage();

// 1. Navigation Tabs — use saved order (includes renames) or fall back to defaults
const activeTab = ref('Men');
const tabs = ref<string[]>(_saved.allTabs.length ? [..._saved.allTabs] : [...DEFAULT_TABS]);

// sellerCreatedTabs: from storage + default empty tabs still present in current tabs
const sellerCreatedTabs = ref<Set<string>>(new Set([
  ..._saved.sellerTabs,
  ...DEFAULT_SELLER_TABS.filter(t => tabs.value.includes(t)),
]));

// Auto-Center Tabs
const tabRefs = ref<HTMLElement[]>([]);
const selectTab = (tab: string, index: number) => {
  activeTab.value = tab;
  const selectedEl = tabRefs.value[index];
  if (selectedEl) {
    selectedEl.scrollIntoView({ behavior: 'smooth', block: 'nearest', inline: 'center' });
  }
};

// --- VISUAL ENGINE ---
const gradients = [
  'bg-gradient-to-br from-orange-50 to-orange-100',
  'bg-gradient-to-br from-blue-50 to-blue-100',
  'bg-gradient-to-br from-purple-50 to-purple-100',
  'bg-gradient-to-br from-emerald-50 to-emerald-100',
  'bg-gradient-to-br from-rose-50 to-rose-100',
  'bg-gradient-to-br from-amber-50 to-yellow-100',
  'bg-gradient-to-br from-indigo-50 to-violet-100',
  'bg-gradient-to-br from-slate-50 to-gray-100',
];
const getCardStyle = (index: number) => ({ gradient: gradients[index % gradients.length] });

// Seller Form toggle
const showForm = ref(false);

// Handle seller submission
function onAddItem({ category, item }: { category: string; item: any }) {
  if (!allData.value[category]) {
    allData.value[category] = [];
    tabs.value.push(category);
    sellerCreatedTabs.value.add(category);
  }
  allData.value[category].push(item);
  activeTab.value = category;
  persistCategory(category);
  console.log('[Collage] Current category data structure:', JSON.parse(JSON.stringify(allData.value)));
}

function onDeleteItem({ category, id }: { category: string; id: string }) {
  if (!allData.value[category]) return;
  allData.value[category] = allData.value[category].filter(i => i.id !== id);
  persistCategory(category);
  console.log(`[Collage] Item deleted: id=${id} from "${category}"`);
}

function onUpdateItem({ category, item }: { category: string; item: any }) {
  if (!allData.value[category]) return;
  const idx = allData.value[category].findIndex(i => i.id === item.id);
  if (idx !== -1) allData.value[category][idx] = item;
  persistCategory(category);
  console.log(`[Collage] Item updated in "${category}":`, item);
}

function onRenameCategory({ oldName, newName }: { oldName: string; newName: string }) {
  if (!newName.trim() || oldName === newName) return;
  // Use full object replacement so Vue detects the key change reactively
  const entries = Object.entries(allData.value);
  const renamed: Record<string, any[]> = {};
  for (const [k, v] of entries) {
    renamed[k === oldName ? newName : k] = v;
  }
  allData.value = renamed;
  // Update tabs list
  const idx = tabs.value.indexOf(oldName);
  if (idx !== -1) tabs.value[idx] = newName;
  // Update sellerCreatedTabs if applicable
  if (sellerCreatedTabs.value.has(oldName)) {
    sellerCreatedTabs.value.delete(oldName);
    sellerCreatedTabs.value.add(newName);
  }
  // Update active tab if needed
  if (activeTab.value === oldName) activeTab.value = newName;
  persistSeller();
  console.log(`[Collage] Category renamed: "${oldName}" → "${newName}"`);
}

// --- DATA SOURCE ---
// Separate storage key for static category overrides
const STATIC_KEY = 'collage_static_overrides';
function loadStaticOverrides(): Record<string, any[]> {
  try {
    const raw = localStorage.getItem(STATIC_KEY);
    if (raw) return JSON.parse(raw);
  } catch {}
  return {};
}
function saveStaticOverride(category: string, items: any[]) {
  try {
    const overrides = loadStaticOverrides();
    overrides[category] = items;
    localStorage.setItem(STATIC_KEY, JSON.stringify(overrides));
  } catch {}
}

// Static API data first, saved seller categories then static overrides on top
const allData = ref<Record<string, any[]>>({
  Men: [
    { id: 'm1', title: 'Sweatpants', image: 'https://images.unsplash.com/photo-1552902865-b72c031ac5ea?auto=format&fit=crop&q=80&w=500' },
    { id: 'm2', title: 'T-Shirts', image: 'https://images.unsplash.com/photo-1521572163474-6864f9cf17ab?auto=format&fit=crop&q=80&w=500' },
    { id: 'm3', title: 'Hoodies', full: true, image: 'https://images.unsplash.com/photo-1556906781-9a412961c28c?auto=format&fit=crop&q=80&w=500' },
    { id: 'm4', title: 'Jeans', image: 'https://images.unsplash.com/photo-1542272454315-4c01d7abdf4a?auto=format&fit=crop&q=80&w=500' },
    { id: 'm5', title: 'Sneakers', image: 'https://images.unsplash.com/photo-1549298916-b41d501d3772?auto=format&fit=crop&q=80&w=500' },
    { id: 'm6', title: 'Watches', image: 'https://images.unsplash.com/photo-1524592094714-0f0654e20314?auto=format&fit=crop&q=80&w=500' },
    { id: 'm7', title: 'Jackets', image: 'https://images.unsplash.com/photo-1591047139829-d91aecb6caea?auto=format&fit=crop&q=80&w=500' },
  ],
  Women: [
    { id: 'w1', title: 'Dresses', image: 'https://images.unsplash.com/photo-1595777457583-95e059d581b8?auto=format&fit=crop&q=80&w=500' },
    { id: 'w2', title: 'Skirts', image: 'https://images.unsplash.com/photo-1583496661160-fb5886a0aaaa?auto=format&fit=crop&q=80&w=500' },
    { id: 'w3', title: 'Tops', image: 'https://images.unsplash.com/photo-1434389677669-e08b4cac3105?auto=format&fit=crop&q=80&w=500' },
    { id: 'w4', title: 'Heels', full: true, image: 'https://images.unsplash.com/photo-1543163521-1bf539c55dd2?auto=format&fit=crop&q=80&w=500' },
    { id: 'w5', title: 'Bags', image: 'https://images.unsplash.com/photo-1584917865442-de89df76afd3?auto=format&fit=crop&q=80&w=500' },
    { id: 'w6', title: 'Jewelry', image: 'https://images.unsplash.com/photo-1515562141207-7a88fb7ce338?auto=format&fit=crop&q=80&w=500' },
    { id: 'w7', title: 'Activewear', image: 'https://images.unsplash.com/photo-1518310383802-640c2de311b2?auto=format&fit=crop&q=80&w=500' },
    { id: 'w8', title: 'Sunglasses', image: 'https://images.unsplash.com/photo-1511499767150-a48a237f0083?auto=format&fit=crop&q=80&w=500' },
  ],
  Beauty: [
    { id: 'b1', title: 'Perfume', image: 'https://images.unsplash.com/photo-1541643600914-78b084683601?auto=format&fit=crop&q=80&w=500' },
    { id: 'b2', title: 'Lipstick', image: 'https://images.unsplash.com/photo-1586495777744-4413f21062fa?auto=format&fit=crop&q=80&w=500' },
    { id: 'b3', title: 'Skincare', image: 'https://images.unsplash.com/photo-1570172619644-dfd03ed5d881?auto=format&fit=crop&q=80&w=500' },
    { id: 'b4', title: 'Palettes', image: 'https://images.unsplash.com/photo-1512496015851-a90fb38ba796?auto=format&fit=crop&q=80&w=500' },
    { id: 'b5', title: 'Serums', image: 'https://images.unsplash.com/photo-1620916566398-39f1143ab7be?auto=format&fit=crop&q=80&w=500' },
  ],
  Kids: [
    { id: 'k1', title: 'Toys', image: 'https://images.unsplash.com/photo-1596461404969-9ae70f2830c1?auto=format&fit=crop&q=80&w=500' },
    { id: 'k2', title: 'School', image: 'https://images.unsplash.com/photo-1503919545885-d94c035542be?auto=format&fit=crop&q=80&w=500' },
    { id: 'k3', title: 'Baby Wear', image: 'https://images.unsplash.com/photo-1522771930-78848d9293e8?auto=format&fit=crop&q=80&w=500' },
    { id: 'k4', title: 'Sneakers', image: 'https://images.unsplash.com/photo-1514989940723-e8a51630c71f?auto=format&fit=crop&q=80&w=500' },
  ],
  Home: [
    { id: 'h1', title: 'Decor', image: 'https://images.unsplash.com/photo-1513519245088-0e12902e5a38?auto=format&fit=crop&q=80&w=500' },
    { id: 'h2', title: 'Bedding', image: 'https://images.unsplash.com/photo-1631679706909-1844bbd07221?auto=format&fit=crop&q=80&w=500' },
    { id: 'h3', title: 'Plants', image: 'https://images.unsplash.com/photo-1485955900006-10f4d324d411?auto=format&fit=crop&q=80&w=500' },
    { id: 'h4', title: 'Chairs', image: 'https://images.unsplash.com/photo-1598300042247-d088f8ab3a91?auto=format&fit=crop&q=80&w=500' },
    { id: 'h5', title: 'Lighting', image: 'https://images.unsplash.com/photo-1507473888900-52e1adad5420?auto=format&fit=crop&q=80&w=500' },
  ],
  himanshu: [],
  gooo: [],
  ..._saved.categories,       // seller-created tabs (restored)
  ...loadStaticOverrides(),   // static tab overrides (items added/edited/deleted on Men, Women etc.)
});

function persistSeller() {
  const sellerOnly: Record<string, any[]> = {};
  sellerCreatedTabs.value.forEach(t => { sellerOnly[t] = allData.value[t] ?? []; });
  saveToStorage(sellerOnly, [...sellerCreatedTabs.value], [...tabs.value]);
}

function persistCategory(category: string) {
  if (sellerCreatedTabs.value.has(category)) {
    persistSeller();
  } else {
    saveStaticOverride(category, allData.value[category] ?? []);
  }
}

// --- DATA PREPARATION ---

// 1. Mobile Calculator (Your existing 60/40 logic)
const calculateMobileLayout = (items: any[]) => {
  let pairIndex = 0;
  return items.map((item, index) => {
    let widthClass = '';
    if (item.full) {
      widthClass = 'w-full';
      pairIndex = 0;
    } else {
      const isStart = pairIndex % 2 === 0;
      const row = Math.floor(pairIndex / 2);
      widthClass = (row % 2 === 0)
        ? (isStart ? 'w-[calc(60%-0.375rem)]' : 'w-[calc(40%-0.375rem)]')
        : (isStart ? 'w-[calc(40%-0.375rem)]' : 'w-[calc(60%-0.375rem)]');
      pairIndex++;
    }
    // Mobile Height is Fixed per row type to keep it clean
    const heightClass = 'h-48 sm:h-56';
    return { ...item, mobileClass: widthClass, heightClass, globalIndex: index };
  });
};

// 2. PC Masonry Distributor (Splits items into 4 columns for gapless layout)
const masonryColumns = computed(() => {
  const cols = [[], [], [], []] as any[][];
  
  // We process the raw displayedItems
  displayedItems.value.forEach((item, i) => {
    const colIndex = i % 4; // Distribute: 0, 1, 2, 3, 0...
    
    // PC Zigzag Logic: Alternate heights within the column
    // Even items in column = Medium (22rem)
    // Odd items in column = Tall (28rem)
    // (We offset cols 2 & 4 to create the wave)
    let hClass = '';
    const positionInCol = Math.floor(i / 4);

    if (colIndex % 2 === 0) {
       // Cols 1 & 3: Start Short, then Tall
       hClass = positionInCol % 2 === 0 ? 'h-[22rem]' : 'h-[28rem]';
    } else {
       // Cols 2 & 4: Start Tall, then Short (Creates offset wave)
       hClass = positionInCol % 2 === 0 ? 'h-[28rem]' : 'h-[22rem]';
    }

    cols[colIndex].push({ 
        ...item, 
        pcHeight: hClass,
        globalIndex: i 
    });
  });
  
  return cols;
});

// --- INFINITE SCROLL LOGIC ---
const displayedItems = ref<any[]>([]);
const isLoadingMore = ref(false);
const loadTrigger = ref<HTMLElement | null>(null);
// Track snapshot length of source data to detect seller add/delete/edit
const sourceSnapshot = ref(0);

const loadMoreData = async () => {
  if (isLoadingMore.value) return;
  const categoryData = allData.value[activeTab.value] || [];
  if (!categoryData.length) return;
  isLoadingMore.value = true;
  await new Promise(resolve => setTimeout(resolve, 800));
  // Re-read in case tab changed during wait
  const freshData = allData.value[activeTab.value] || [];
  const newBatch = freshData.map(item => ({
    ...item,
    id: item.id + '_' + Date.now() + Math.random()
  }));
  displayedItems.value = [...displayedItems.value, ...newBatch];
  isLoadingMore.value = false;
  // If trigger is still visible after loading, keep filling (handles small datasets)
  await nextTick();
  if (loadTrigger.value) {
    const rect = loadTrigger.value.getBoundingClientRect();
    if (rect.top < window.innerHeight) loadMoreData();
  }
};

const mobileItems = computed(() => calculateMobileLayout(displayedItems.value));

watch(activeTab, () => {
  displayedItems.value = [];
  sourceSnapshot.value = (allData.value[activeTab.value] || []).length;
  loadMoreData();
}, { immediate: true });

// Only reset when source data length changes (add/delete) — not on every deep mutation
watch(() => (allData.value[activeTab.value] || []).length, (newLen) => {
  if (newLen !== sourceSnapshot.value) {
    sourceSnapshot.value = newLen;
    displayedItems.value = [];
    loadMoreData();
  }
});

onMounted(() => {
  const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) loadMoreData();
  }, { rootMargin: '100px', threshold: 0.1 });
  if (loadTrigger.value) observer.observe(loadTrigger.value);
  watch(loadTrigger, (el) => { if (el) observer.observe(el); });
});
</script>

<template>
  <div class="min-h-screen bg-gray-50 dark:bg-black text-gray-900 dark:text-gray-100 font-sans selection:bg-pink-500 selection:text-white">

    <!-- Seller Form Panel -->
    <div class="max-w-[1920px] mx-auto px-4 md:px-8 pt-4">
      <button
        @click="showForm = !showForm"
        class="mb-3 px-5 py-2 rounded-full bg-black dark:bg-white text-white dark:text-black text-xs font-bold tracking-wider hover:opacity-70 transition"
      >
        {{ showForm ? '✕ Close Seller Form' : '+ Seller: Add Item' }}
      </button>
      <Transition name="slide">
        <SellerForm
          v-if="showForm"
          :allData="allData"
          :tabs="tabs"
          :sellerCreatedTabs="sellerCreatedTabs"
          @add-item="onAddItem"
          @delete-item="onDeleteItem"
          @update-item="onUpdateItem"
          @rename-category="onRenameCategory"
          class="mb-6"
        />
      </Transition>
    </div>

    <nav :class="['bg-white/80 dark:bg-black/80 backdrop-blur-xl border-b border-gray-200 dark:border-gray-800 transition-all duration-300', mode === 'sticky' ? 'sticky top-0 z-50' : 'relative']">
      <div class="max-w-[1920px] mx-auto px-4 md:px-8">
        <div class="flex flex-col md:flex-row md:items-center md:justify-between py-4 gap-4">
          <div class="overflow-x-auto no-scrollbar w-full flex md:justify-center">
            <div class="flex px-1 gap-4 md:gap-12 min-w-max md:min-w-0 md:justify-center">
              <button
                v-for="(tab, index) in tabs"
                :key="tab"
                :ref="el => { if(el) tabRefs[index] = el as HTMLElement }"
                @click="selectTab(tab, index)"
                class="relative px-2 py-4 text-sm md:text-lg font-bold transition-all duration-300 group outline-none focus-visible:ring-2 focus-visible:ring-pink-500 rounded-lg"
                :class="activeTab === tab ? 'text-black dark:text-white' : 'text-gray-400 hover:text-gray-600 dark:hover:text-gray-300'"
              >
                {{ tab }}
                <span class="absolute bottom-2 left-0 w-full h-1 rounded-full transition-all duration-300 origin-center" :class="activeTab === tab ? 'bg-black dark:bg-white scale-x-100' : 'bg-transparent scale-x-0 group-hover:bg-gray-300 group-hover:scale-x-50'"></span>
              </button>
            </div>
          </div>
        </div>
      </div>
    </nav>

    <main class="px-3 py-4 md:px-8 md:py-10 max-w-[1920px] mx-auto">
      
      <div class="flex flex-wrap gap-3 md:hidden">
        <TransitionGroup name="stagger">
          <div
            v-for="(item) in mobileItems"
            :key="item.id"
            class="relative group cursor-pointer overflow-hidden rounded-2xl
                   shadow-sm hover:shadow-2xl
                   bg-white dark:bg-gray-900
                   transition-all duration-500 ease-out hover:-translate-y-1"
            :class="[
              item.mobileClass,
              item.heightClass,
              getCardStyle(item.globalIndex).gradient
            ]"
          >
             <img :src="item.image" loading="lazy" class="absolute inset-0 w-full h-full object-cover" alt="Item" />
             <div class="absolute inset-0 bg-gradient-to-t from-black/50 via-transparent to-transparent opacity-60"></div>
             <div class="absolute bottom-3 left-3 bg-white/95 dark:bg-black/80 backdrop-blur-md px-3 py-1.5 rounded-full shadow-lg z-10">
                <span class="text-[10px] font-bold uppercase tracking-widest text-black dark:text-white">{{ item.title }}</span>
             </div>
          </div>
        </TransitionGroup>
      </div>


      <div class="hidden md:flex md:gap-6 items-start">
        
        <div v-for="(colItems, colIndex) in masonryColumns" :key="colIndex" class="flex-1 flex flex-col gap-6">
           <TransitionGroup name="stagger">
             <div
               v-for="(item) in colItems"
               :key="item.id"
               class="relative group cursor-pointer overflow-hidden rounded-3xl
                      shadow-sm hover:shadow-2xl
                      bg-white dark:bg-gray-900 w-full
                      transition-all duration-500 ease-out hover:-translate-y-1"
               :class="[
                 item.pcHeight,  // Zigzag Height (22rem or 28rem)
                 getCardStyle(item.globalIndex).gradient
               ]"
             >
                <img 
                  :src="item.image" 
                  loading="lazy" 
                  class="absolute inset-0 w-full h-full object-cover transform group-hover:scale-110 transition-all duration-700 ease-[cubic-bezier(0.25,1,0.5,1)]" 
                  alt="Item" 
                />
                
                <div class="absolute inset-0 bg-gradient-to-t from-black/50 via-transparent to-transparent opacity-60"></div>
                
                <div class="absolute bottom-5 left-5 bg-white/95 dark:bg-black/80 backdrop-blur-md px-5 py-2.5 rounded-full shadow-lg z-10 group-hover:bg-black group-hover:text-white dark:group-hover:bg-white dark:group-hover:text-black transition-colors duration-300">
                   <span class="text-xs font-bold uppercase tracking-widest">{{ item.title }}</span>
                </div>
             </div>
           </TransitionGroup>
        </div>

      </div>

      <div ref="loadTrigger" class="h-24 w-full flex items-center justify-center mt-10">
        <div v-if="isLoadingMore" class="flex gap-2">
            <span class="w-3 h-3 bg-pink-500 rounded-full animate-bounce"></span>
            <span class="w-3 h-3 bg-pink-500 rounded-full animate-bounce delay-100"></span>
            <span class="w-3 h-3 bg-pink-500 rounded-full animate-bounce delay-200"></span>
        </div>
      </div>

    </main>
  </div>
</template>

<style scoped>
.no-scrollbar::-webkit-scrollbar { display: none; }
.no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }

.stagger-move,
.stagger-enter-active,
.stagger-leave-active { transition: all 0.5s cubic-bezier(0.55, 0, 0.1, 1); }
.stagger-enter-from,
.stagger-leave-to { opacity: 0; transform: scale(0.9) translateY(20px); }

.slide-enter-active, .slide-leave-active { transition: all 0.35s ease; }
.slide-enter-from, .slide-leave-to { opacity: 0; transform: translateY(-12px); }
</style>