<template>
  <n-layout content-style="padding: 24px;">
  <div class="media-edit-page">
    <n-page-header @back="goBack">
      <template #title>
        手动编辑媒体演员信息
      </template>
    </n-page-header>

    <n-divider />

    <div v-if="isLoading" class="loading-container">
      <n-spin size="large" />
      <p style="text-align: center; margin-top: 10px;">正在加载媒体详情...</p>
    </div>

    <div v-else-if="itemDetails && itemDetails.item_name">
      <n-grid cols="1 l:4" :x-gap="24" responsive="screen">
        <!-- 左侧信息栏 (海报) -->
        <n-grid-item span="1 l:1">
          <n-space vertical :size="24">
            <n-card :title="itemDetails.item_name" :bordered="false">
              <template #cover>
                <n-image
                  :src="posterUrl"
                  lazy
                  object-fit="cover"
                  class="media-poster"
                >
                  <template #placeholder>
                    <div class="poster-placeholder">
                      <n-icon :component="ImageIcon" size="48" :depth="3" />
                    </div>
                  </template>
                </n-image>
              </template>
              <template #header-extra>
                <n-tag :type="itemDetails.item_type === 'Movie' ? 'info' : 'success'" size="small" round>
                  {{ itemTypeInChinese }}
                </n-tag>
              </template>
              <n-descriptions label-placement="left" bordered :column="1" size="small">
                <n-descriptions-item label="Emby ItemID">
                  {{ itemDetails.item_id }}
                </n-descriptions-item>
                <n-descriptions-item label="原始记录评分" v-if="itemDetails.original_score !== null && itemDetails.original_score !== undefined">
                  <n-tag type="warning" size="small">{{ itemDetails.original_score }}</n-tag>
                </n-descriptions-item>
                <n-descriptions-item label="待复核原因" v-if="itemDetails.review_reason">
                  <n-text type="error">{{ itemDetails.review_reason }}</n-text>
                </n-descriptions-item>
              </n-descriptions>
            </n-card>

            <n-card title="辅助工具" :bordered="false">
              <n-space vertical>
                <n-form-item label="数据操作" label-placement="top">
                  <n-space>
                    <n-button 
                      tag="a" 
                      :href="searchLinks.google_search_wiki"
                      target="_blank" 
                      :disabled="!searchLinks.google_search_wiki"
                      :loading="isLoading"
                    >
                      Google搜索
                    </n-button>
                    <n-button
                      type="info"
                      @click="translateAllFields" 
                      :loading="isTranslating" 
                      :disabled="isLoading"
                    >
                      一键翻译
                    </n-button>
                  </n-space>
                </n-form-item>
                <n-form-item label="从URL提取" label-placement="top">
                  <n-input-group>
                    <n-input 
                      v-model:value="urlToParse" 
                      placeholder="粘贴维基百科演员表URL"
                      clearable
                    />
                    <n-button 
                      type="primary" 
                      @click="parseCastFromUrl" 
                      :loading="isParsingFromUrl"
                      :disabled="!urlToParse"
                    >
                      提取
                    </n-button>
                  </n-input-group>
                  <template #feedback>
                    <n-text depth="3" style="font-size: 0.85em;">
                      提取后将自动与右方列表进行匹配更新。
                    </n-text>
                  </template>
                </n-form-item>
              </n-space>
            </n-card>
          </n-space>
        </n-grid-item>

        <!-- 右侧演员列表 -->
        <n-grid-item span="1 l:3">
          <n-card title="演员列表 (可拖动排序)" :bordered="false">
            
            <n-form label-placement="left" label-width="auto">
              <draggable
                v-model="editableCast"
                tag="div"
                item-key="_temp_id"
                class="actor-grid-container"
                handle=".drag-handle"
                animation="300"
              >
                <template #item="{ element: actor, index }">
                  <div class="actor-card-header">
                    <n-card size="small" class="actor-edit-card" content-style="padding: 16px;">
                      <template #header>
                        <div class="actor-card-header">
                          <n-avatar
                            round
                            size="small"
                            :style="{ backgroundColor: getAvatarColor(actor.name) }"
                          >
                            {{ index + 1 }}
                          </n-avatar>
                          <span class="actor-name-title" :title="actor.name">{{ actor.name || '新演员' }}</span>
                        </div>
                      </template>
                      <template #header-extra>
                        <n-space>
                          <n-button text class="drag-handle">
                            <n-icon :component="DragHandleIcon" />
                          </n-button>
                          <n-popconfirm @positive-click="removeActor(index)">
                            <template #trigger>
                              <n-button text type="error">
                                <n-icon :component="TrashIcon" />
                              </n-button>
                            </template>
                            确定要删除演员 “{{ actor.name || '新演员' }}” 吗？
                          </n-popconfirm>
                        </n-space>
                      </template>
                      
                      <div class="actor-card-content">
                        <n-image
                          :src="getActorImageUrl(actor)"
                          lazy
                          object-fit="cover"
                          class="actor-avatar-image"
                        >
                          <template #placeholder>
                            <div class="avatar-placeholder">
                              <n-icon :component="PersonIcon" size="24" :depth="3" />
                            </div>
                          </template>
                        </n-image>
                        
                        <div class="actor-inputs">
                          <n-form-item label="演员" label-placement="left" label-width="40" class="compact-form-item">
                            <n-input v-model:value="actor.name" placeholder="演员名" size="small" style="width: 100%;" />
                          </n-form-item>
                          <n-form-item label="角色" label-placement="left" label-width="40" class="compact-form-item">
                            <n-input v-model:value="actor.role" placeholder="角色名" size="small" style="width: 100%;" />
                          </n-form-item>
                        </div>
                      </div>
                    </n-card>
                  </div>
                </template>
              </draggable>
            </n-form>
            
            <div class="sticky-actions">
              <n-space>
                <n-button @click="goBack">返回列表</n-button>
                <n-button type="primary" @click="handleSaveChanges" :loading="isSaving">
                  保存修改
                </n-button>
              </n-space>
            </div>
          </n-card>
        </n-grid-item>
      </n-grid>
    </div>

    <div v-else class="error-container">
      <n-alert title="错误" type="error">
        无法加载媒体详情，或指定的媒体项不存在。请检查后端日志或确认该媒体项有效。
        <n-button text @click="goBack" style="margin-left: 10px;">返回列表</n-button>
      </n-alert>
    </div>
  </div>
  </n-layout>
</template>

<script setup>
import { ref, onMounted, watch, computed, nextTick } from 'vue';
import draggable from 'vuedraggable';
import { NIcon, NInput, NInputGroup, NGrid, NGridItem, NFormItem, NTag, NAvatar, NPopconfirm, NImage } from 'naive-ui';
import { useRoute, useRouter } from 'vue-router';
import axios from 'axios';
import { NPageHeader, NDivider, NSpin, NCard, NDescriptions, NDescriptionsItem, NButton, NSpace, NAlert, useMessage } from 'naive-ui';
import {
  MoveOutline as DragHandleIcon,
  TrashOutline as TrashIcon,
  ImageOutline as ImageIcon,
  PersonOutline as PersonIcon
} from '@vicons/ionicons5';

const route = useRoute();
const router = useRouter();
const message = useMessage();

const itemId = ref(null);
const isLoading = ref(true);
const itemDetails = ref(null);
const editableCast = ref([]);
const isSaving = ref(false);

const searchLinks = ref({ google_search_wiki: '' });
const isParsingFromUrl = ref(false);
const urlToParse = ref('');
const isTranslating = ref(false);

const posterUrl = computed(() => {
  if (itemDetails.value?.item_id && itemDetails.value?.image_tag) {
    return `/image_proxy/Items/${itemDetails.value.item_id}/Images/Primary?tag=${itemDetails.value.image_tag}&quality=90`;
  }
  return '';
});

const getActorImageUrl = (actor) => {
  return actor.imageUrl || ''; 
};

const itemTypeInChinese = computed(() => {
  if (!itemDetails.value || !itemDetails.value.item_type) {
    return '';
  }
  switch (itemDetails.value.item_type) {
    case 'Movie':
      return '电影';
    case 'Series':
      return '电视剧';
    default:
      return itemDetails.value.item_type;
  }
});

const getAvatarColor = (name) => {
  const colors = ['#f56a00', '#7265e6', '#ffbf00', '#00a2ae', '#4caf50', '#2196f3'];
  if (!name || name.length === 0) return colors[0];
  let hash = 0;
  for (let i = 0; i < name.length; i++) {
    hash = name.charCodeAt(i) + ((hash << 5) - hash);
  }
  const index = Math.abs(hash % colors.length);
  return colors[index];
};

watch(() => itemDetails.value, (newItemDetails) => {
  if (newItemDetails?.current_emby_cast) {
    editableCast.value = newItemDetails.current_emby_cast.map((actor, index) => ({
      ...actor,
      _temp_id: `actor-${Date.now()}-${index}`,
    }));
  } else {
    editableCast.value = [];
  }
}, { deep: true });

const removeActor = (index) => {
  editableCast.value.splice(index, 1);
  message.info("已从编辑列表移除一个演员（尚未保存）。");
};


const parseCastFromUrl = async () => {
  if (!urlToParse.value.trim()) {
    message.warning("请输入要解析的URL。");
    return;
  }
  isParsingFromUrl.value = true;
  try {
    const response = await axios.post('/api/parse_cast_from_url', { url: urlToParse.value });
    const newCastFromWeb = response.data;

    if (newCastFromWeb && Array.isArray(newCastFromWeb) && newCastFromWeb.length > 0) {
      message.success(`成功提取 ${newCastFromWeb.length} 位演员信息，正在与当前列表进行匹配更新...`);
      handleEnrich(newCastFromWeb);
    } else {
      message.info(response.data.message || "未从该URL中找到有效的演员信息。");
    }
  } catch (error) {
    console.error("从URL解析演员失败:", error);
    message.error(error.response?.data?.error || "解析失败，请检查URL或后端日志。");
  } finally {
    isParsingFromUrl.value = false;
  }
};

const handleEnrich = async (newCastFromWeb) => {
  try {
    const payload = {
      current_cast: editableCast.value,
      new_cast_from_web: newCastFromWeb
    };
    const response = await axios.post('/api/actions/enrich_cast_list', payload);
    const enrichedList = response.data;

    editableCast.value = enrichedList.map((actor, index) => ({
      ...actor,
      _temp_id: `enriched-actor-${Date.now()}-${index}`
    }));
    
    message.success("列表已更新！请检查匹配结果并保存。");
    urlToParse.value = '';

  } catch (error) {
    console.error("补充列表失败:", error);
    message.error(error.response?.data?.error || "更新列表时发生错误，请检查后端日志。");
  }
};

const translateAllFields = async () => {
  // ...
  try {
    // 【★★★ 构建包含所有上下文的最终 Payload ★★★】
    const payload = { 
      cast: editableCast.value,
      title: itemDetails.value.item_name,
      year: itemDetails.value.production_year,
    };

    const response = await axios.post('/api/actions/translate_cast_sa', payload);
    const translatedList = response.data;

    editableCast.value = translatedList.map((actor, index) => ({
      ...actor,
      _temp_id: `translated-actor-${Date.now()}-${index}`
    }));
    
    message.success("智能翻译完成！");

  } catch (error) {
    console.error("一键翻译失败:", error);
    message.error(error.response?.data?.error || "翻译失败，请检查后端日志。");
  } finally {
    isTranslating.value = false;
  }
};

const fetchMediaDetails = async () => {
  isLoading.value = true;
  try {
    const response = await axios.get(`/api/media_for_editing_sa/${itemId.value}`);
    itemDetails.value = response.data;

    // ★★★ 核心修复：在这里添加下面这行代码 ★★★
    // 检查后端返回的数据中是否有 search_links，如果有，就更新它
    if (response.data && response.data.search_links) {
      searchLinks.value = response.data.search_links;
    }
    // ★★★ 修复结束 ★★★

  } catch (error) {
    message.error(error.response?.data?.error || "获取媒体详情失败。");
    itemDetails.value = null;
  } finally {
    isLoading.value = false;
  }
};

onMounted(() => {
  itemId.value = route.params.itemId;
  
  if (itemId.value) {
    console.log(`准备为 Item ID: ${itemId.value} 获取详情...`);
    fetchMediaDetails();
  } else {
    message.error("未提供媒体项ID！");
    isLoading.value = false;
  }
});

const goBack = () => {
  router.push({ name: 'ReviewList' });
};

const handleSaveChanges = async () => {
  if (!itemDetails.value?.item_id) return;
  isSaving.value = true;
  try {
    // 等待任何可能的输入框更新完成
    await nextTick();

    // ★★★ 核心修复：明确构建发送到后端的演员对象结构 ★★★
    const castPayload = editableCast.value.map(actor => {
      // 从前端的 actor 对象中提取需要的数据，并使用后端期望的键名
      return {
        tmdbId: actor.tmdbId, // 确保发送 tmdbId
        name: actor.name,     // 发送 name
        role: actor.role      // 发送 role
      };
    });

    const payload = {
      cast: castPayload,
      item_name: itemDetails.value.item_name,
    };
    
    // (可选) 在发送前打印最终的 payload，用于调试
    console.log("----------- [最终发送到后端的数据] -----------");
    console.log(JSON.stringify(payload, null, 2));

    await axios.post(`/api/update_media_cast_sa/${itemDetails.value.item_id}`, payload);
    
    message.success("修改已保存，Emby将自动刷新。");
    // 延迟一小段时间再返回，给用户反馈时间
    setTimeout(() => {
      goBack();
    }, 1500);

  } catch (error) {
    console.error("保存修改失败:", error);
    message.error(error.response?.data?.error || "保存修改失败，请检查后端日志。");
  } finally {
    isSaving.value = false;
  }
};
</script>

<style scoped>
.media-edit-page {
  padding: 0 24px 24px 24px;
  transition: all 0.3s;
}

.loading-container, .error-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 200px;
}

.media-poster {
  width: 100%;
  height: auto;
  background-color: var(--n-card-color);
  aspect-ratio: 2 / 3;
}

.poster-placeholder, .avatar-placeholder {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: var(--n-action-color);
}

.actor-grid-container {
  display: grid;
  grid-template-columns: repeat(1, 1fr);
  gap: 16px;
}
/* ★★★ START: 核心修改 - 移除 2xl 的 5 列规则 ★★★ */
@media (min-width: 640px) { /* s */
  .actor-grid-container { grid-template-columns: repeat(2, 1fr); }
}
@media (min-width: 768px) { /* m */
  .actor-grid-container { grid-template-columns: repeat(2, 1fr); }
}
@media (min-width: 1024px) { /* l */
  .actor-grid-container { grid-template-columns: repeat(3, 1fr); }
}
@media (min-width: 1280px) { /* xl */
  .actor-grid-container { grid-template-columns: repeat(4, 1fr); }
}
/* 移除了针对 1536px 以上屏幕的 5 列规则 */
/* ★★★ END: 核心修改 ★★★ */

.actor-edit-card:hover {
  transform: translateY(-4px);
  box-shadow: var(--n-box-shadow-hover) !important;
}

.actor-card-header {
  display: flex;
  align-items: center;
  gap: 8px;
}

.actor-name-title {
  font-weight: 600;
  flex-grow: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.actor-card-content {
  display: flex;
  align-items: center;
  gap: 12px;
}

.actor-avatar-image {
  width: 100px;
  height: 100px;
  border-radius: var(--n-border-radius);
  flex-shrink: 0;
}

.actor-inputs {
  display: flex;
  flex-direction: column;
  gap: 8px;
  flex-grow: 1;
  flex-basis: 0;
  min-width: 0;
}

.compact-form-item {
  margin-bottom: 0 !important;
}

.sticky-actions {
  position: sticky;
  bottom: -24px;
  left: 0;
  right: 0;
  padding: 16px 24px;
  background-color: var(--n-color);
  border-top: 1px solid var(--n-border-color);
  display: flex;
  justify-content: flex-end;
  z-index: 10;
  margin: 24px -24px 0;
}

.drag-handle {
  cursor: grab;
}
.drag-handle:active {
  cursor: grabbing;
}

.sortable-ghost {
  opacity: 0.4;
  background: var(--n-action-color);
  border: 1px dashed var(--n-border-color);
}
.sortable-drag {
  opacity: 1 !important;
  transform: rotate(2deg);
  box-shadow: 0 10px 20px rgba(0,0,0,0.2);
  z-index: 99;
}
</style>