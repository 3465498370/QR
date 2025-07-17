 <template>
  <div class="qr-generator">
    <h2>二维码生成器</h2>
    <form @submit.prevent="generateQr">
      <label>类型：</label>
      <select v-model="form.type">
        <option value="text">文本</option>
        <option value="url">网址/链接</option>
        <option value="image">图片</option>
        <option value="video">视频</option>
      </select>
      <br /><br />
      <label>内容：</label>
      <input v-if="form.type === 'text' || form.type === 'url'" v-model="form.content" placeholder="请输入内容" />
      <div v-if="form.type === 'image' || form.type === 'video'" class="upload-area" :aria-busy="uploading"
           @dragover.prevent
           @drop.prevent="onDropFile"
           @click="onClickUpload">
        <span v-if="!fileName">拖拽或点击上传{{ form.type === 'image' ? '图片' : '视频' }}</span>
        <span v-else>{{ fileName }}</span>
        <input ref="fileInput" type="file" :accept="acceptType" style="display:none" @change="onFileChange" />
        <div v-if="uploading" class="uploading-tip">
          正在上传... {{ uploadPercent }}%
        </div>
        <div v-if="uploadSuccess" class="upload-success">上传成功！</div>
      </div>
      <br />
      <button type="submit">生成二维码</button>
    </form>
    <div v-if="qrValue" class="qr-result">
      <qrcode-vue :value="qrValue" :size="200" />
      <br />
      <button @click="downloadQr">下载二维码</button>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, computed } from 'vue'
import QrcodeVue from 'qrcode.vue'
import OSS from 'ali-oss'

const form = ref({
  type: 'text',
  content: ''
})
const qrValue = ref('')
const fileInput = ref(null)
const fileName = ref('')
const fileUrl = ref('')
const uploading = ref(false)
const uploadPercent = ref(0)
const uploadSuccess = ref(false)

const ossConfig = {
  region: import.meta.env.VITE_OSS_REGION,
  accessKeyId: import.meta.env.VITE_OSS_ACCESS_KEY_ID,
  accessKeySecret: import.meta.env.VITE_OSS_ACCESS_KEY_SECRET,
  bucket: import.meta.env.VITE_OSS_BUCKET
}

const acceptType = computed(() => {
  if (form.value.type === 'image') return 'image/*'
  if (form.value.type === 'video') return 'video/*'
  return ''
})

watch(() => form.value.type, () => {
  form.value.content = ''
  fileName.value = ''
  fileUrl.value = ''
  uploadPercent.value = 0
  uploadSuccess.value = false
})

function onClickUpload() {
  fileInput.value && fileInput.value.click()
}

function onFileChange(e) {
  const file = e.target.files[0]
  handleFile(file)
}

function onDropFile(e) {
  const file = e.dataTransfer.files[0]
  handleFile(file)
}

async function handleFile(file) {
  if (!file) return
  fileName.value = file.name
  uploading.value = true
  uploadPercent.value = 0
  uploadSuccess.value = false
  try {
    const client = new OSS(ossConfig)
    const ext = file.name.substring(file.name.lastIndexOf('.'))
    const ossFileName = `qr-upload/${Date.now()}_${Math.random().toString(36).slice(2)}${ext}`
    const result = await client.multipartUpload(ossFileName, file, {
      progress: (p) => {
        uploadPercent.value = Math.floor(p * 100)
      }
    })
    fileUrl.value = result.url
    form.value.content = fileUrl.value
    uploadSuccess.value = true
  } catch (e) {
    alert('上传失败: ' + (e.message || e.toString()))
  }
  uploading.value = false
}

function generateQr() {
  if ((form.value.type === 'text' || form.value.type === 'url') && !form.value.content) {
    alert('请输入内容')
    return
  }
  if ((form.value.type === 'image' || form.value.type === 'video') && !form.value.content) {
    alert('请上传文件')
    return
  }
  // 生成二维码内容为Vercel生产环境地址
  const base = 'https://qr-gsd7.vercel.app/'
  const url = `${base}redirect.html?type=${encodeURIComponent(form.value.type)}&content=${encodeURIComponent(form.value.content)}`
  qrValue.value = url
}

function downloadQr() {
  const canvas = document.querySelector('.qr-result canvas')
  if (canvas) {
    const url = canvas.toDataURL('image/png')
    const a = document.createElement('a')
    a.href = url
    a.download = 'qrcode.png'
    a.click()
  }
}
</script>

<style scoped>
.qr-generator {
  max-width: 400px;
  margin: 40px auto;
  padding: 24px;
  border: 1px solid #eee;
  border-radius: 8px;
  background: #fff;
}
.qr-result {
  text-align: center;
  margin-top: 20px;
}
.upload-area {
  width: 100%;
  min-height: 60px;
  border: 1px dashed #aaa;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  background: #fafafa;
  color: #888;
  margin-bottom: 10px;
  position: relative;
}
.upload-area[aria-busy="true"] {
  opacity: 0.6;
  pointer-events: none;
}
.uploading-tip {
  position: absolute;
  left: 0; right: 0; top: 0; bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(255,255,255,0.7);
  color: #409eff;
  font-size: 16px;
  z-index: 2;
}
.upload-success {
  color: #52c41a;
  font-size: 14px;
  margin-top: 4px;
}
</style>
