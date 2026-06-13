<template>
  <div>
    <div style="margin-bottom:24px;">
      <h1 style="color:white;margin-bottom:8px;">交换记录</h1>
      <p style="color:rgba(255,255,255,0.8);">查看你完成的所有交换</p>
    </div>

    <div v-if="loading" style="text-align:center;padding:60px;color:white;">
      加载中...
    </div>

    <div v-else-if="exchanges.length === 0" class="empty-state card">
      <h2>暂无交换记录</h2>
      <p style="margin-bottom:20px;">去市场看看，寻找心仪的盲盒吧！</p>
      <router-link to="/">
        <button class="btn btn-primary">浏览盲盒市场</button>
      </router-link>
    </div>

    <div v-else class="grid" style="gap:20px;">
      <div v-for="exc in exchanges" :key="exc.id" class="card">
        <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:16px;">
          <span class="badge badge-exchanged" style="font-size:12px;">交换成功</span>
          <span style="color:#999;font-size:13px;">{{ formatDate(exc.createdAt) }}</span>
        </div>

        <div style="display:flex;gap:24px;align-items:center;">
          <div style="flex:1;text-align:center;">
            <p style="color:#667eea;font-weight:600;margin-bottom:8px;">我的物品</p>
            <div v-if="exc.myItem" style="display:flex;gap:12px;align-items:center;justify-content:center;">
              <img :src="appendAuth(exc.myItem.image)"
                   style="width:80px;height:80px;object-fit:cover;border-radius:8px;"/>
              <div style="text-align:left;">
                <p style="font-weight:500;">{{ exc.myItem.realName }}</p>
                <p style="color:#999;font-size:12px;">
                  {{ exc.myItem.mysteryTags ? exc.myItem.mysteryTags.join(', ') : '' }}
                </p>
              </div>
            </div>
          </div>

          <div style="font-size:32px;color:#667eea;">
            &lt;&gt;
          </div>

          <div style="flex:1;text-align:center;">
            <p style="color:#f5576c;font-weight:600;margin-bottom:8px;">对方物品</p>
            <div v-if="exc.otherItem" style="display:flex;gap:12px;align-items:center;justify-content:center;">
              <img :src="appendAuth(exc.otherItem.image)"
                   style="width:80px;height:80px;object-fit:cover;border-radius:8px;"/>
              <div style="text-align:left;">
                <p style="font-weight:500;">{{ exc.otherItem.realName }}</p>
                <p style="color:#999;font-size:12px;">
                  {{ exc.otherItem.ownerName }}
                </p>
              </div>
            </div>
          </div>
        </div>

        <div v-if="exc.otherContact"
             style="margin-top:20px;padding:16px;background:#eef2ff;border-radius:8px;">
          <p style="font-weight:500;margin-bottom:8px;">对方联系方式</p>
          <p style="color:#667eea;font-size:18px;font-weight:600;">{{ exc.otherContact }}</p>
          <p style="color:#999;font-size:12px;margin-top:8px;">请主动联系对方完成物品交换</p>
        </div>

        <div style="margin-top:20px;padding:16px;background:linear-gradient(135deg,#fff9e6 0%,#fff3cd 100%);border-radius:8px;border:1px dashed #ffc107;">
          <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:12px;">
            <div style="display:flex;align-items:center;gap:8px;">
              <span style="font-size:18px;">📝</span>
              <span style="font-weight:600;color:#856404;">私人小纸条</span>
              <span style="font-size:11px;color:#856404;opacity:0.7;background:#ffeeba;padding:2px 6px;border-radius:10px;">仅自己可见</span>
            </div>
            <div>
              <template v-if="!editingNote[exc.id] && exc.myNote">
                <button class="btn-link" @click="startEditNote(exc)" style="margin-right:8px;">编辑</button>
                <button class="btn-link btn-link-danger" @click="handleDeleteNote(exc)">删除</button>
              </template>
              <template v-else-if="!editingNote[exc.id]">
                <button class="btn-link" @click="startEditNote(exc)">添加备注</button>
              </template>
            </div>
          </div>

          <template v-if="editingNote[exc.id]">
            <textarea
              v-model="noteDraft[exc.id]"
              rows="3"
              placeholder="记录一些只有你自己能看到的内容..."
              style="width:100%;padding:10px;border:1px solid #ffc107;border-radius:6px;resize:vertical;font-size:14px;background:#fff;color:#333;box-sizing:border-box;font-family:inherit;"
            ></textarea>
            <div style="display:flex;justify-content:flex-end;gap:8px;margin-top:10px;">
              <button class="btn btn-secondary" style="padding:6px 14px;font-size:13px;" @click="cancelEditNote(exc)">取消</button>
              <button class="btn btn-primary" style="padding:6px 14px;font-size:13px;" @click="handleSaveNote(exc)" :disabled="savingNote[exc.id]">
                {{ savingNote[exc.id] ? '保存中...' : '保存' }}
              </button>
            </div>
          </template>
          <template v-else>
            <div v-if="exc.myNote" style="color:#856404;font-size:14px;line-height:1.6;white-space:pre-wrap;">{{ exc.myNote }}</div>
            <div v-else style="color:#856404;opacity:0.6;font-size:13px;font-style:italic;">暂无备注，点击"添加备注"记录一下吧~</div>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue'
import { getMyExchanges, appendAuth, saveExchangeNote, deleteExchangeNote } from '../api/index.js'
import { userStore } from '../store/user.js'

const exchanges = ref([])
const loading = ref(true)
const editingNote = reactive({})
const noteDraft = reactive({})
const savingNote = reactive({})

function formatDate(dateStr) {
  const date = new Date(dateStr)
  const y = date.getFullYear()
  const m = String(date.getMonth() + 1).padStart(2, '0')
  const d = String(date.getDate()).padStart(2, '0')
  const h = String(date.getHours()).padStart(2, '0')
  const min = String(date.getMinutes()).padStart(2, '0')
  return y + '-' + m + '-' + d + ' ' + h + ':' + min
}

async function loadExchanges() {
  loading.value = true
  try {
    exchanges.value = await getMyExchanges(userStore.user.id)
  } catch (e) {
    alert('加载失败')
  } finally {
    loading.value = false
  }
}

function startEditNote(exc) {
  editingNote[exc.id] = true
  noteDraft[exc.id] = exc.myNote || ''
}

function cancelEditNote(exc) {
  editingNote[exc.id] = false
  noteDraft[exc.id] = ''
}

async function handleSaveNote(exc) {
  const content = (noteDraft[exc.id] || '').trim()
  if (!content) {
    alert('备注内容不能为空')
    return
  }
  savingNote[exc.id] = true
  try {
    const res = await saveExchangeNote(exc.id, userStore.user.id, content)
    exc.myNote = res.note.content
    editingNote[exc.id] = false
    noteDraft[exc.id] = ''
  } catch (e) {
    alert('保存失败：' + (e.response?.data?.error || e.message))
  } finally {
    savingNote[exc.id] = false
  }
}

async function handleDeleteNote(exc) {
  if (!confirm('确定要删除这条私人备注吗？')) return
  try {
    await deleteExchangeNote(exc.id, userStore.user.id)
    exc.myNote = null
  } catch (e) {
    alert('删除失败：' + (e.response?.data?.error || e.message))
  }
}

onMounted(loadExchanges)
</script>
