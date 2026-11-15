<script setup>
import { ref } from 'vue'

const docType = ref('INVOICE')
const selectedFile = ref(null)
const previewUrl = ref(null)
const fileInput = ref(null)
const xmlResult = ref('')
const isLoading = ref(false)
const errorMessage = ref('')
const conversionTime = ref(null)

// ចាប់យក File ពេល Select ឬ Drop
const handleFileChange = (e) => {
  const file = e.target.files ? e.target.files[0] : e.dataTransfer.files[0]
  if (!file) return

  selectedFile.value = file
  errorMessage.value = ''
  xmlResult.value = ''
  conversionTime.value = null

  // បង្កើត Preview បើជាប្រភេទរូបភាព
  if (previewUrl.value) URL.revokeObjectURL(previewUrl.value)
  if (file.type.startsWith('image/')) {
    previewUrl.value = URL.createObjectURL(file)
  } else {
    previewUrl.value = null
  }
}

// ផ្លាស់ប្តូរ File ថ្មី
const changeFile = () => {
  selectedFile.value = null
  if (previewUrl.value) URL.revokeObjectURL(previewUrl.value)
  previewUrl.value = null
  xmlResult.value = ''
  conversionTime.value = null
  errorMessage.value = ''
  if (fileInput.value) {
    fileInput.value.value = ''
    fileInput.value.click()
  }
}

// ផ្ញើ File ទៅកាន់ FastAPI Backend
const convertDocument = async () => {
  if (!selectedFile.value) {
    errorMessage.value = 'Please choose a file first!'
    return
  }

  isLoading.value = true
  errorMessage.value = ''
  conversionTime.value = null

  const startTime = performance.now()

  const formData = new FormData()
  formData.append('file', selectedFile.value)
  formData.append('doc_type', docType.value)

  try {
    const response = await fetch('https://backendpermitconvertfilexml.onrender.com/', {
      method: 'POST',
      body: formData,
    })

    const data = await response.json()

    if (data.status === 'success') {
      xmlResult.value = data.xml_data
    } else {
      errorMessage.value = data.error || 'Conversion failed!'
    }
  } catch (err) {
    errorMessage.value = 'Failed to connect to the server (FastAPI)!'
  } finally {
    const endTime = performance.now()
    conversionTime.value = ((endTime - startTime) / 1000).toFixed(2)
    isLoading.value = false
  }
}

// ទាញយក File .xml
const downloadXml = () => {
  const blob = new Blob([xmlResult.value], { type: 'application/xml' })
  const link = document.createElement('a')
  link.href = URL.createObjectURL(blob)
  link.download = `${docType.value.toLowerCase()}_converted.xml`
  link.click()
  URL.revokeObjectURL(link.href)
}
</script>

<template>
  <div class="min-h-screen bg-slate-100 p-6 flex flex-col items-center">
    <div class="w-full max-w-5xl bg-white rounded-2xl shadow-sm border border-slate-200 p-8">
      
      <!-- Header -->
      <header class="mb-6 border-b border-slate-100 pb-4">
        <h1 class="text-2xl font-bold text-slate-800">Trade Document to XML</h1>
        <p class="text-sm text-slate-500">Read and Convert to Permit, Invoice, Packing List, B/L ទៅជា XML Auto</p>
      </header>

      <div class="grid grid-cols-1 md:grid-cols-2 gap-8">
        <!-- ផ្នែកខាងឆ្វេង: Controls & Upload -->
        <div class="space-y-4">
          <div v-if="conversionTime !== null" class="text-xs text-slate-500 bg-slate-100 px-2 py-0.5 rounded-md inline-block">
            Time to convert: {{ conversionTime }} seconds
          </div>

          <!-- Drag and Drop Box -->
          <div 
            @dragover.prevent 
            @drop.prevent="handleFileChange"
            class="border-2 border-dashed border-slate-300 rounded-xl p-6 flex flex-col items-center justify-center text-center cursor-pointer hover:border-blue-500 bg-slate-50 transition"
          >
            <input ref="fileInput" type="file" id="file" accept="image/*,application/pdf" class="hidden" @change="handleFileChange" />
            <label for="file" class="cursor-pointer">
              <div class="text-slate-600 text-sm font-medium">Click here to dropdown File (Drag & Drop)</div>
              <div class="text-xs text-slate-400 mt-1">Supported formats: PNG, JPG, JPEG, PDF</div>
            </label>
            <div v-if="selectedFile" class="mt-3 flex items-center gap-2 text-xs font-semibold text-blue-600 bg-blue-50 px-3 py-1 rounded-full">
              <span>{{ selectedFile.name }}</span>
              <button type="button" @click.stop="changeFile" class="text-blue-700 underline hover:text-blue-900">
                 Change File
              </button>
            </div>
          </div>

          <!-- Image Preview -->
          <div v-if="previewUrl" class="border rounded-lg p-2 bg-slate-50 max-h-48 overflow-hidden flex justify-center">
            <img :src="previewUrl" alt="Preview" class="object-contain max-h-44 rounded" />
          </div>

          <button 
            @click="convertDocument" 
            :disabled="isLoading || !selectedFile"
            class="w-full bg-blue-600 text-white font-medium py-2.5 rounded-lg hover:bg-blue-700 transition disabled:bg-slate-300 cursor-pointer"
          >
            <span v-if="isLoading">Reading file...</span>
            <span v-else>Start Conversion</span>
          </button>

          <p v-if="errorMessage" class="text-sm text-red-600 bg-red-50 p-3 rounded-lg">{{ errorMessage }}</p>
        </div>

        <!-- ផ្នែកខាងស្តាំ: XML Result Preview -->
        <div class="flex flex-col h-full">
          <div class="flex justify-between items-center mb-2">
            <span class="text-sm font-semibold text-slate-700">The result XML:</span>
            <button 
              v-if="xmlResult" 
              @click="downloadXml" 
              class="bg-emerald-600 text-white text-xs px-3 py-1.5 rounded-md hover:bg-emerald-700 transition"
            >
              Download .xml
            </button>
          </div>

          <textarea 
            :value="xmlResult" 
            placeholder="XML code will be displayed here after conversion..." 
            class="w-full flex-1 min-h-[300px] font-mono text-xs bg-slate-900 text-emerald-400 p-4 rounded-xl border border-slate-800 resize-none outline-none"
            readonly
          ></textarea>
        </div>
      </div>

    </div>
  </div>
</template>
