<script setup>
import { NSpace, NAlert, NSwitch, NCard, NInput, NInputGroupLabel } from 'naive-ui'
import { NButton, NLayout, NInputGroup, NModal, NSelect, NPagination } from 'naive-ui'
import { NList, NListItem, NThing, NTag, NIcon, NSplit, NResult } from 'naive-ui'
import { NDrawer, NDrawerContent } from 'naive-ui'
import { watch, onMounted, ref } from "vue";
import { useMessage } from 'naive-ui'
import { useI18n } from 'vue-i18n'
import { useGlobalState } from '../store'
import { api } from '../api'
import { CloudDownloadRound } from '@vicons/material'
import { useIsMobile } from '../utils/composables'

const message = useMessage()
const isMobile = useIsMobile()

const { settings, themeSwitch } = useGlobalState()
const autoRefresh = ref(false)
const data = ref([])
const timer = ref(null)

const count = ref(0)
const page = ref(1)
const pageSize = ref(20)

const showAttachments = ref(false)
const curAttachments = ref([])
const curMail = ref(null);

const { t } = useI18n({
  locale: 'zh',
  messages: {
    en: {
      autoRefresh: 'Auto Refresh',
      refresh: 'Refresh',
      attachments: 'Show Attachments',
      pleaseSelectMail: "Please select a mail to view."
    },
    zh: {
      autoRefresh: '自动刷新',
      refresh: '刷新',
      attachments: '查看附件',
      pleaseSelectMail: "请选择一封邮件查看。"
    }
  }
});

const setupAutoRefresh = async (autoRefresh) => {
  if (autoRefresh) {
    timer.value = setInterval(async () => {
      await refresh();
    }, 30000)
  } else {
    clearInterval(timer.value)
    timer.value = null
  }
}

watch(autoRefresh, async (autoRefresh, old) => {
  setupAutoRefresh(autoRefresh)
})

watch([page, pageSize], async ([page, pageSize], [oldPage, oldPageSize]) => {
  if (page !== oldPage || pageSize !== oldPageSize) {
    await refresh();
  }
})

const refresh = async () => {
  if (typeof settings.value.address != 'string' || settings.value.address.trim() === '') {
    return;
  }
  try {
    const { results, count: totalCount } = await api.fetch(
      `/api/mails`
      + `?limit=${pageSize.value}`
      + `&offset=${(page.value - 1) * pageSize.value}`
    );
    data.value = results;
    if (totalCount > 0) {
      count.value = totalCount;
    }
    if (!isMobile.value && !curMail.value && data.value.length > 0) {
      curMail.value = results[0];
    }
  } catch (error) {
    message.error(error.message || "error");
    console.error(error);
  }
};

const clickRow = async (row) => {
  curMail.value = row;
};

const getAttachments = async (attachment_id) => {
  try {
    const res = await api.fetch(
      `/api/attachment/${attachment_id}`
    );
    curAttachments.value = res
      .filter((item) => item?.content?.data)
      .map((item) => {
        return {
          id: item.contentId || Math.random().toString(36).substring(2, 15),
          filename: item.filename || "",
          size: item.size,
          url: URL.createObjectURL(
            new Blob(
              [new Uint8Array(item.content.data)],
              { type: item.contentType || 'application/octet-stream' }
            ))
        }
      });
    showAttachments.value = true;
  } catch (error) {
    message.error(error.message || "error");
  }
};

const mailItemClass = (row) => {
  return curMail.value && row.id == curMail.value.id ? (themeSwitch.value ? 'overlay overlay-dark-backgroud' : 'overlay overlay-light-backgroud') : '';
};

onMounted(async () => {
  await api.getSettings();
  await refresh();
});
</script>

<template>
  <div>
    <n-layout v-if="settings.address">
      <n-split class="left" v-if="!isMobile" direction="horizontal" :max="0.75" :min="0.25" :default-size="0.25">
        <template #1>
          <div>
            <div style="display: inline-block; margin-top: 10px; margin-bottom: 10px;">
              <n-pagination v-model:page="page" v-model:page-size="pageSize" :item-count="count" simple size="small" />
            </div>
            <n-switch v-model:value="autoRefresh" size="small">
              <template #checked>
                {{ t('autoRefresh') }}
              </template>
              <template #unchecked>
                {{ t('autoRefresh') }}
              </template></n-switch>
            <n-button class="center" @click="refresh" size="small" type="primary">
              {{ t('refresh') }}
            </n-button>
          </div>
          <div style="overflow: scroll; max-height: 80vh;">
            <n-list hoverable clickable>
              <n-list-item v-for="row in data" v-bind:key="row.id" @click="() => clickRow(row)"
                :class="mailItemClass(row)">
                <n-thing class="center" :title="row.subject" style="overflow: scroll">
                  <template #description>
                    <n-tag type="info">
                      ID: {{ row.id }}
                    </n-tag>
                    <n-tag type="info">
                      {{ row.created_at }}
                    </n-tag>
                    <div style="word-break: break-all; font-size: small;">
                      FROM: {{ row.source }}
                    </div>
                  </template>
                </n-thing>
              </n-list-item>
            </n-list>
          </div>
        </template>
        <template #2>
          <n-card v-if="curMail" :title="curMail.subject" style="overflow: scroll;">
            <n-space>
              <n-tag type="info">
                ID: {{ curMail.id }}
              </n-tag>
              <n-tag type="info">
                {{ curMail.created_at }}
              </n-tag>
              <n-tag type="info">
                FROM: {{ curMail.source }}
              </n-tag>
              <n-button v-if="curMail.attachment_id" size="small" tertiary type="info"
                @click="getAttachments(curMail.attachment_id)">
                {{ t('attachments') }}
              </n-button>
            </n-space>
            <div v-html="curMail.message" style="max-height: 100vh;"></div>
          </n-card>
          <n-card v-else>
            <n-result status="info" :title="t('pleaseSelectMail')">
            </n-result>
          </n-card>
        </template>
      </n-split>
      <div class="left" v-else>
        <div>
          <div style="display: inline-block; margin-top: 10px; margin-bottom: 10px;">
            <n-pagination v-model:page="page" v-model:page-size="pageSize" :item-count="count" simple size="small" />
          </div>
          <n-switch v-model:value="autoRefresh" size="small">
            <template #checked>
              {{ t('autoRefresh') }}
            </template>
            <template #unchecked>
              {{ t('autoRefresh') }}
            </template></n-switch>
          <n-button class="center" @click="refresh" size="small" type="primary">
            {{ t('refresh') }}
          </n-button>
        </div>
        <div id="drawer-target" style="overflow: scroll; max-height: 80vh;">
          <n-list hoverable clickable>
            <n-list-item v-for="row in data" v-bind:key="row.id" @click="() => clickRow(row)">
              <n-thing class="center" :title="row.subject" style="overflow: scroll">
                <template #description>
                  <n-tag type="info">
                    ID: {{ row.id }}
                  </n-tag>
                  <n-tag type="info">
                    {{ row.created_at }}
                  </n-tag>
                  <div style="word-break: break-all; font-size: small;">
                    FROM: {{ row.source }}
                  </div>
                </template>
              </n-thing>
            </n-list-item>
          </n-list>
        </div>
        <n-drawer v-model:show="curMail" width="100%" :trap-focus="false" :block-scroll="false" to="#drawer-target">
          <n-drawer-content :title="curMail.subject" closable>
            <n-card style="overflow: scroll;">
              <n-space>
                <n-tag type="info">
                  ID: {{ curMail.id }}
                </n-tag>
                <n-tag type="info">
                  {{ curMail.created_at }}
                </n-tag>
                <n-tag type="info">
                  FROM: {{ curMail.source }}
                </n-tag>
                <n-button v-if="curMail.attachment_id" size="small" tertiary type="info"
                  @click="getAttachments(curMail.attachment_id)">
                  {{ t('attachments') }}
                </n-button>
              </n-space>
              <div v-html="curMail.message" style="max-height: 100vh;"></div>
            </n-card>
          </n-drawer-content>
        </n-drawer>
      </div>
    </n-layout>
    <n-modal v-model:show="showAttachments" preset="dialog" title="Dialog">
      <template #header>
        <div>{{ t("attachments") }}</div>
      </template>
      <n-list hoverable clickable>
        <n-list-item v-for="row in curAttachments" v-bind:key="row.id">
          <n-thing class="center" :title="row.filename">
            <template #description>
              <n-space>
                <n-tag type="info">
                  Size: {{ row.size }}
                </n-tag>
              </n-space>
            </template>
          </n-thing>
          <template #suffix>
            <n-button tag="a" target="_blank" tertiary type="info" size="small'" :download="row.filename"
              :href="row.url">
              <n-icon :component="CloudDownloadRound" />
            </n-button>
          </template>
        </n-list-item>
      </n-list>
      <template #action>
      </template>
    </n-modal>
  </div>
</template>

<style scoped>
.left {
  overflow: scroll;
  text-align: left;
}

.overlay {
  width: 100%;
  height: 100%;
  z-index: 1000;
}

.overlay-dark-backgroud {
  background-color: rgba(255, 255, 255, 0.1);
}

.overlay-light-backgroud {
  background-color: rgba(0, 0, 0, 0.1);
}
</style>
