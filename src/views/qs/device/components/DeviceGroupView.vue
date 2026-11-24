<template>
  <div class="device-group-view" v-loading="loading">
    <div class="group-summary" v-if="devices.length">
      {{ nvrCount }} 台录像机 · {{ videoTotal }} 路通道 ·
      <em v-if="videoOffline">{{ videoOffline }} 路离线</em>
      <span v-else>全部在线</span>
    </div>

    <div v-for="g in groups" :key="g.key" class="group" :class="{ collapsed: !isOpen(g) }">
      <div class="g-head" @click="toggle(g)">
        <el-icon class="arrow" :class="{ open: isOpen(g) }"><ArrowRight /></el-icon>
        <span class="dot" :class="g.online ? 'on' : 'off'"></span>
        <span class="g-name">{{ g.name }}</span>
        <span class="g-meta">{{ g.meta }}</span>
        <span class="spacer"></span>
        <span class="g-count" :class="{ warn: g.onlineCount < g.videos.length }">
          <b>{{ g.onlineCount }}</b> / {{ g.videos.length }} 在线
        </span>
        <el-dropdown v-if="g.gbDeviceId" trigger="click" @command="(c: string) => handleGroupCommand(c, g)">
          <span class="icon-btn" @click.stop><el-icon><MoreFilled /></el-icon></span>
          <template #dropdown>
            <el-dropdown-menu>
              <el-dropdown-item command="refresh" :icon="Refresh">同步通道</el-dropdown-item>
              <el-dropdown-item command="add" :icon="Plus">添加通道</el-dropdown-item>
            </el-dropdown-menu>
          </template>
        </el-dropdown>
      </div>

      <template v-if="isOpen(g)">
        <div class="grid">
          <div
            v-for="ch in shownVideos(g)"
            :key="ch.row.id"
            class="ch"
            :class="{ off: !isOnline(ch.row), checked: ch.row.checked }"
            @click="handleCardClick(ch.row)"
          >
            <template v-if="isOnline(ch.row)">
              <img v-if="ch.row.snap" class="scene" :src="ch.row.snap" alt="" />
              <div v-else class="scene empty"><el-icon><Picture /></el-icon></div>
              <div class="shade"></div>
              <div class="play"><span><el-icon v-if="ch.row.loading" class="is-loading"><Loading /></el-icon><el-icon v-else><CaretRight /></el-icon></span></div>
              <div v-if="ch.row.streamStatus === '1'" class="badge"><i></i>直播中</div>
            </template>
            <div v-else class="offline"><el-icon><VideoCamera /></el-icon>离线</div>

            <div class="check" @click.stop>
              <el-checkbox v-model="ch.row.checked" @change="emit('selection-change')" />
            </div>

            <div class="tools" @click.stop>
              <el-tooltip v-if="ch.row.streamStatus === '1' && ch.row.type !== '13'" content="停止">
                <span @click="emit('stop', ch.row)"><el-icon><SwitchButton /></el-icon></span>
              </el-tooltip>
              <el-tooltip v-if="ch.row.type === '13'" content="接入地址">
                <span @click="emit('access-address', ch.row)"><el-icon><Position /></el-icon></span>
              </el-tooltip>
              <el-tooltip content="云端录像">
                <span @click="emit('cloud-record', ch.row)"><el-icon><Film /></el-icon></span>
              </el-tooltip>
              <el-dropdown trigger="click" @command="(c: string) => emit('more', c, ch.row)">
                <span><el-icon><MoreFilled /></el-icon></span>
                <template #dropdown>
                  <el-dropdown-menu>
                    <el-dropdown-item command="edit" :icon="Edit">编辑</el-dropdown-item>
                    <el-dropdown-item v-if="!['1', '2', '3', '4', '6', '13'].includes(ch.row.type || '')" command="__deviceRecord" :icon="Monitor">设备录像</el-dropdown-item>
                    <el-dropdown-item command="viewSnapshots" :icon="Picture">查看抓图</el-dropdown-item>
                    <el-dropdown-item command="__toggleStatus" :icon="SwitchButton">{{ ch.row.status === 'ENABLE' ? '停用' : '启用' }}</el-dropdown-item>
                    <el-dropdown-item command="delete" :icon="Delete" divided>删除</el-dropdown-item>
                  </el-dropdown-menu>
                </template>
              </el-dropdown>
            </div>

            <div class="info">
              <span class="nm" :title="ch.row.deviceName">{{ ch.row.deviceName }}</span>
              <span v-if="ch.row.status !== 'ENABLE'" class="disabled-tag">已停用</span>
              <span class="no">{{ ch.label }}</span>
            </div>
          </div>
        </div>

        <div v-if="g.videos.length > PAGE" class="more" @click="toggleExpand(g)">
          {{ expanded[g.key] ? '收起' : `查看全部 ${g.videos.length} 路` }}
        </div>

        <div v-if="g.audios.length" class="audio">
          <span>音频输出</span>
          <el-tooltip v-for="a in g.audios" :key="a.id" :content="a.gbChannelId || ''" placement="top">
            <span class="pill" :class="{ off: !isOnline(a) }" @click="emit('more', 'edit', a)">
              <el-icon><Microphone /></el-icon>{{ a.deviceName }}
            </span>
          </el-tooltip>
        </div>
      </template>
    </div>

    <el-empty v-if="!loading && !groups.length" description="暂无设备数据" />
  </div>
</template>

<script setup lang="ts" name="DeviceGroupView">
import { computed, reactive, ref, watch } from 'vue'
import { ElMessage } from 'element-plus'
import {
  ArrowRight, MoreFilled, Refresh, Plus, Picture, Loading, CaretRight, VideoCamera,
  SwitchButton, Position, Film, Edit, Monitor, Delete, Microphone
} from '@element-plus/icons-vue'
import { getAllDevices } from '@/api/qs/gb28181'
import type { Gb28181Device } from '@/types/api/qs/gb28181'
import type { QsDevice } from '@/types/api/qs/device'

type Row = QsDevice & { checked?: boolean; loading?: boolean }

interface ChannelItem {
  row: Row
  label: string
}

interface Group {
  key: string
  name: string
  meta: string
  online: boolean
  gbDeviceId?: string
  videos: ChannelItem[]
  audios: Row[]
  onlineCount: number
}

const props = defineProps<{
  devices: Row[]
  loading?: boolean
  liveStreamTypes?: { label: string; value: string }[]
}>()

const emit = defineEmits<{
  (e: 'play', row: Row): void
  (e: 'stop', row: Row): void
  (e: 'cloud-record', row: Row): void
  (e: 'access-address', row: Row): void
  (e: 'more', command: string, row: Row): void
  (e: 'selection-change'): void
  (e: 'refresh', row: Row): void
  (e: 'add-channel', gbDeviceId: string): void
}>()

/** 每组默认展示的通道数，超出折叠 */
const PAGE = 8
/** GB28181 编码第 11-13 位为 137 表示音频输出通道 */
const AUDIO_OUT_TYPE = '137'
const DIRECT_KEY = '__direct__'

const gbDevices = ref<Gb28181Device[]>([])
const openState = reactive<Record<string, boolean>>({})
const expanded = reactive<Record<string, boolean>>({})

const isOnline = (row: Row) => row.deviceStatus === 'ON'
const isAudioOut = (row: Row) => !!row.gbChannelId && row.gbChannelId.substring(10, 13) === AUDIO_OUT_TYPE

function loadGbDevices() {
  getAllDevices().then(res => {
    gbDevices.value = res.data || []
  }).catch(() => {})
}
// 设备列表刷新时同步刷新国标设备（录像机）信息
watch(() => props.devices, loadGbDevices, { immediate: true })

const typeLabel = (type?: string) => props.liveStreamTypes?.find(d => d.value === type)?.label || ''

const groups = computed<Group[]>(() => {
  const map = new Map<string, Row[]>()
  for (const row of props.devices) {
    const key = row.type === '12' && row.gbDeviceId ? row.gbDeviceId : DIRECT_KEY
    if (!map.has(key)) map.set(key, [])
    map.get(key)!.push(row)
  }

  const result: Group[] = []
  map.forEach((rows, key) => {
    const isDirect = key === DIRECT_KEY
    const gb = isDirect ? undefined : gbDevices.value.find(d => d.deviceId === key)
    const videoRows = rows.filter(r => !isAudioOut(r))
      .sort((a, b) => String(a.gbChannelId ?? a.channel ?? a.id).localeCompare(String(b.gbChannelId ?? b.channel ?? b.id)))
    const videos = videoRows.map((row, i) => ({
      row,
      label: isDirect
        ? (typeLabel(row.type) || row.ipAddress || '')
        : 'CH' + String(i + 1).padStart(2, '0')
    }))
    const onlineCount = videoRows.filter(isOnline).length
    result.push({
      key,
      gbDeviceId: isDirect ? undefined : key,
      name: isDirect ? '独立设备' : (gb?.name || key),
      meta: isDirect ? '未接入录像机的摄像机' : [gb?.ip || rows[0]?.ipAddress, '国标28181', gb?.name ? key : ''].filter(Boolean).join(' · '),
      online: isDirect ? onlineCount > 0 : (gb ? !!gb.onLine : rows.some(isOnline)),
      videos,
      audios: rows.filter(isAudioOut),
      onlineCount
    })
  })
  // 录像机在前，独立设备放最后
  return result.sort((a, b) => Number(a.key === DIRECT_KEY) - Number(b.key === DIRECT_KEY))
})

const nvrCount = computed(() => groups.value.filter(g => g.gbDeviceId).length)
const videoTotal = computed(() => groups.value.reduce((s, g) => s + g.videos.length, 0))
const videoOffline = computed(() => groups.value.reduce((s, g) => s + g.videos.length - g.onlineCount, 0))

/** 离线的录像机默认收起，其它默认展开；用户手动切换后保持 */
const isOpen = (g: Group) => openState[g.key] ?? g.online
const toggle = (g: Group) => { openState[g.key] = !isOpen(g) }
const toggleExpand = (g: Group) => { expanded[g.key] = !expanded[g.key] }
const shownVideos = (g: Group) => expanded[g.key] ? g.videos : g.videos.slice(0, PAGE)

/** 点卡片 = 播放；离线时只提示，编辑统一走右上角「…」菜单 */
function handleCardClick(row: Row) {
  if (isOnline(row)) emit('play', row)
  else ElMessage.warning('设备离线，无法播放')
}

function handleGroupCommand(command: string, g: Group) {
  if (command === 'refresh') {
    const row = g.videos[0]?.row || g.audios[0]
    if (row) emit('refresh', row)
  } else if (command === 'add' && g.gbDeviceId) {
    emit('add-channel', g.gbDeviceId)
  }
}
</script>

<style lang="scss" scoped>
.device-group-view {
  min-height: 200px;
}

.group-summary {
  font-size: 13px;
  color: #8f959e;
  margin: 4px 2px 14px;

  em {
    font-style: normal;
    color: #f54a45;
  }
}

.group {
  background: var(--el-bg-color);
  border: 1px solid var(--el-border-color-lighter);
  border-radius: 14px;
  padding: 6px 20px 20px;
  margin-bottom: 16px;

  &.collapsed {
    padding-bottom: 6px;
  }
}

.g-head {
  display: flex;
  align-items: center;
  gap: 10px;
  height: 56px;
  cursor: pointer;
  user-select: none;

  .arrow {
    color: #b6bac1;
    font-size: 14px;
    transition: transform 0.2s;

    &.open {
      transform: rotate(90deg);
    }
  }

  .spacer {
    flex: 1;
  }
}

.dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex-shrink: 0;

  &.on { background: #34c759; }
  &.off { background: #c4c7cc; }
}

.g-name {
  font-size: 16px;
  font-weight: 600;
  color: var(--el-text-color-primary);
}

.g-meta {
  font-size: 13px;
  color: #8f959e;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.g-count {
  font-size: 13px;
  color: #646a73;
  white-space: nowrap;

  b {
    font-weight: 600;
    color: var(--el-text-color-primary);
  }

  &.warn b {
    color: #f54a45;
  }
}

.icon-btn {
  width: 32px;
  height: 32px;
  border-radius: 8px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  color: #646a73;
  font-size: 16px;
  cursor: pointer;
  transition: background 0.15s;

  &:hover {
    background: var(--el-fill-color-light);
  }
}

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(230px, 1fr));
  gap: 14px;
}

.ch {
  position: relative;
  aspect-ratio: 16 / 10;
  border-radius: 10px;
  overflow: hidden;
  cursor: pointer;
  background: #2b2f36;

  .scene {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    transition: transform 0.4s;

    &.empty {
      display: flex;
      align-items: center;
      justify-content: center;
      color: #5c6370;
      font-size: 28px;
    }
  }

  &:hover .scene {
    transform: scale(1.04);
  }

  .shade {
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    height: 56%;
    background: linear-gradient(to top, rgba(0, 0, 0, 0.62), rgba(0, 0, 0, 0));
  }

  .info {
    position: absolute;
    left: 12px;
    right: 12px;
    bottom: 10px;
    color: #fff;
    display: flex;
    align-items: center;
    gap: 8px;

    .nm {
      font-size: 14px;
      font-weight: 500;
      flex: 1;
      min-width: 0;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }

    .no {
      font-size: 11px;
      opacity: 0.75;
      font-family: "SF Mono", Menlo, Consolas, monospace;
    }

    .disabled-tag {
      font-size: 11px;
      padding: 0 6px;
      border-radius: 8px;
      background: rgba(245, 74, 69, 0.85);
      color: #fff;
    }
  }

  .badge {
    position: absolute;
    top: 10px;
    left: 38px;
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 11px;
    color: #fff;
    background: rgba(0, 0, 0, 0.35);
    padding: 2px 8px 2px 6px;
    border-radius: 10px;

    i {
      width: 6px;
      height: 6px;
      border-radius: 50%;
      background: #ff453a;
      animation: dgv-blink 1.4s infinite;
    }
  }

  .play {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 0;
    transition: opacity 0.2s;

    span {
      width: 44px;
      height: 44px;
      border-radius: 50%;
      background: rgba(255, 255, 255, 0.92);
      color: #1f2329;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 20px;
    }
  }

  .check {
    position: absolute;
    top: 6px;
    left: 10px;
    opacity: 0;
    transition: opacity 0.2s;
  }

  .tools {
    position: absolute;
    top: 8px;
    right: 8px;
    display: flex;
    gap: 4px;
    opacity: 0;
    transition: opacity 0.2s;

    span {
      width: 28px;
      height: 28px;
      border-radius: 7px;
      background: rgba(0, 0, 0, 0.35);
      color: #fff;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 14px;
      cursor: pointer;

      &:hover {
        background: rgba(0, 0, 0, 0.55);
      }
    }
  }

  &:hover {
    .play, .check, .tools { opacity: 1; }
  }

  &.checked {
    box-shadow: 0 0 0 2px var(--el-color-primary);

    .check { opacity: 1; }
  }

  &.off {
    background: var(--el-fill-color-light);

    .offline {
      position: absolute;
      inset: 0;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      gap: 6px;
      color: #b6bac1;
      font-size: 12px;

      .el-icon { font-size: 24px; }
    }

    .info { color: #8f959e; }
    .info .no { opacity: 1; }

    .tools span {
      background: rgba(0, 0, 0, 0.18);
    }
  }
}

.more {
  margin-top: 14px;
  text-align: center;
  font-size: 13px;
  color: var(--el-color-primary);
  cursor: pointer;
}

.audio {
  margin-top: 16px;
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
  font-size: 13px;
  color: #8f959e;

  .pill {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 12px;
    border-radius: 14px;
    background: var(--el-fill-color-lighter);
    color: var(--el-text-color-primary);
    cursor: pointer;
    transition: background 0.15s;

    &:hover {
      background: var(--el-color-primary-light-9);
      color: var(--el-color-primary);
    }

    &.off {
      color: #b6bac1;
    }
  }
}

@keyframes dgv-blink {
  50% { opacity: 0.25; }
}
</style>
