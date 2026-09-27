<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
import { api } from '../api.js'

const role = ref(localStorage.getItem('role') || '')
const gates = ref([])
const events = ref([])
const err = ref('')
let timer

async function refresh() {
  if (!localStorage.getItem('tok')) return
  try {
    const [g, e] = await Promise.all([api('/api/gates'), api('/api/gate-events')])
    gates.value = g
    events.value = e
    err.value = ''
  } catch (e2) {
    err.value = String(e2.message || e2)
  }
}

async function setGate(lamp, action) {
  err.value = ''
  try {
    await api('/api/gates', { method: 'POST', body: JSON.stringify({ lamp, action }) })
    await refresh()
  } catch (e) {
    err.value = String(e.message || e)
  }
}

function fmt(t) {
  if (!t) return ''
  const d = new Date(t)
  return isNaN(d) ? String(t) : d.toLocaleString()
}

function actionLabel(a) {
  return a === 'pause' ? '暂停' : a === 'resume' ? '恢复' : a
}

onMounted(() => {
  role.value = localStorage.getItem('role') || ''
  refresh()
  timer = setInterval(refresh, 1000)
})
onUnmounted(() => clearInterval(timer))
</script>

<template>
  <div>
    <p v-if="err" style="color:#b00020">{{ err }}</p>

    <section style="margin:16px 0; padding:12px; border:1px solid #ccc;">
      <h3>灯种暂停闸</h3>
      <table border="1" cellpadding="6" style="border-collapse:collapse; width:100%;">
        <thead>
          <tr>
            <th>灯种</th><th>闸门状态</th><th>待处理</th><th>最近操作人</th><th>最近操作时间</th>
            <th v-if="role === 'writer'">操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="g in gates" :key="g.lamp">
            <td>{{ g.lamp }}</td>
            <td :style="{ color: g.paused ? '#b00020' : '#1a7f37', fontWeight: 600 }">
              {{ g.paused ? '已暂停' : '正常' }}
            </td>
            <td>{{ g.pending_count }}</td>
            <td>{{ g.updated_by }}</td>
            <td>{{ fmt(g.updated_at) }}</td>
            <td v-if="role === 'writer'">
              <button v-if="!g.paused" type="button" @click="setGate(g.lamp, 'pause')">暂停</button>
              <button v-else type="button" @click="setGate(g.lamp, 'resume')">恢复</button>
            </td>
          </tr>
          <tr v-if="!gates.length">
            <td :colspan="role === 'writer' ? 6 : 5" style="text-align:center; color:#666;">暂无灯种</td>
          </tr>
        </tbody>
      </table>
      <p style="color:#666; font-size:13px;">暂停后领取进程跳过该灯种的待处理任务，其它灯种照常；恢复后继续领取。</p>
    </section>

    <section style="margin:16px 0; padding:12px; border:1px solid #ccc;">
      <h3>闸门流水</h3>
      <table border="1" cellpadding="6" style="border-collapse:collapse; width:100%;">
        <thead>
          <tr>
            <th>编号</th><th>时间</th><th>灯种</th><th>动作</th><th>操作人</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="e in events" :key="e.id">
            <td>{{ e.id }}</td>
            <td>{{ fmt(e.created_at) }}</td>
            <td>{{ e.lamp }}</td>
            <td>{{ actionLabel(e.action) }}</td>
            <td>{{ e.actor }}</td>
          </tr>
          <tr v-if="!events.length">
            <td colspan="5" style="text-align:center; color:#666;">暂无流水</td>
          </tr>
        </tbody>
      </table>
    </section>
  </div>
</template>
