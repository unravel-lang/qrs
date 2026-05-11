<script lang="ts" setup>
import { appendFileHeaderMetaToBuffer } from 'luby-transform'

enum ReadPhase {
  Idle,
  Reading,
  Chunking,
  Ready,
}

type InputMode = 'file' | 'text'

const error = ref<any>()
const fps = ref(20)
const sliceSize = ref(1000)
const throttledFps = useDebounce(fps, 500)
const readPhase = ref<ReadPhase>(ReadPhase.Idle)

const filename = ref<string | undefined>()
const contentType = ref<string | undefined>()
const data = ref<Uint8Array | null>(null)

const inputMode = ref<InputMode>('file')
const textInput = ref('')

const route = useRoute()
const router = useRouter()

onMounted(() => {
  if (route.hash.length > 1) {
    router.replace('/scan')
  }
})

watch(inputMode, () => {
  readPhase.value = ReadPhase.Idle
  data.value = null
  textInput.value = ''
})

async function onFileChange(file?: File) {
  if (!file) {
    readPhase.value = ReadPhase.Idle
    data.value = null
    return
  }

  try {
    readPhase.value = ReadPhase.Reading

    filename.value = file.name
    contentType.value = file.type

    const buffer = await file.arrayBuffer()
    data.value = appendFileHeaderMetaToBuffer(new Uint8Array(buffer), {
      filename: filename.value,
      contentType: contentType.value,
    })

    readPhase.value = ReadPhase.Ready
  }
  catch (e) {
    error.value = e
    readPhase.value = ReadPhase.Idle
    data.value = null
  }
}

function onTextSubmit() {
  const text = textInput.value.trim()
  if (!text)
    return

  const encoder = new TextEncoder()
  data.value = appendFileHeaderMetaToBuffer(encoder.encode(text), {
    filename: 'message.txt',
    contentType: 'text/plain',
  })
  filename.value = 'message.txt'
  contentType.value = 'text/plain'
  readPhase.value = ReadPhase.Ready
}

const config = useRuntimeConfig()

const isPrefixed = ref(true)
const prefix = computed(() => {
  if (!isPrefixed.value) {
    return ''
  }
  if (config.public.qrcodePrefix) {
    return `${config.public.qrcodePrefix}#`
  }
  return `${location.href}${location.pathname}#`
})
</script>

<template>
  <div px="4" flex="~ col-reverse sm:col" h-full w-full gap-4 pb-8 pt-2>
    <div grid="~ cols-1 sm:cols-3 wrap" gap="6 sm:(x-5 y-3)">
      <div flex="~ gap-1" border="~ gray/25 rounded-lg" p-0.5>
        <button
          flex-1 rounded-md px-3 py-1.5 text-sm font-medium transition-colors
          :class="inputMode === 'file' ? 'bg-teal-600 text-white' : 'text-neutral-500 hover:text-neutral-700 dark:hover:text-neutral-300'"
          @click="inputMode = 'file'"
        >
          <span i-carbon:document-add mr-1 inline-block align-text-bottom />
          File
        </button>
        <button
          flex-1 rounded-md px-3 py-1.5 text-sm font-medium transition-colors
          :class="inputMode === 'text' ? 'bg-teal-600 text-white' : 'text-neutral-500 hover:text-neutral-700 dark:hover:text-neutral-300'"
          @click="inputMode = 'text'"
        >
          <span i-carbon:text-align-left mr-1 inline-block align-text-bottom />
          Text
        </button>
      </div>
      <div w-full inline-flex flex-1 flex-row items-center sm:col-span-2>
        <span min-w-32>
          <span pr-2 text-neutral-400>Slice Size</span>
        </span>
        <input
          v-model.lazy="sliceSize"
          type="number"
          :min="1"
          :max="2000"
          border="~ gray/25 rounded-lg"
          w-full flex-1 bg-transparent px2 py1 text-sm shadow-sm
        >
      </div>
      <div>
        <label inline-flex flex-row select-none items-center>
          <span min-w-32>
            <span text-neutral-400>Scanner URL</span>
          </span>
          <InputCheckbox v-model="isPrefixed" />
        </label>
      </div>
      <div w-full inline-flex flex-1 flex-row items-center sm:col-span-2>
        <span min-w-32>
          <span pr-2 text-neutral-400>Ideal FPS</span>
          <span font-mono>{{ throttledFps.toFixed(0) }}hz</span>
        </span>
        <InputSlide
          v-model="throttledFps"
          :min="1"
          :max="60"
          smooth
          w-full flex-1
        />
      </div>
    </div>
    <div
      v-if="readPhase === ReadPhase.Ready && data"
      h-full w-full flex justify-center
    >
      <Generate
        :max-scans-per-second="throttledFps"
        :slice-size="sliceSize"
        :data="data"
        :filename="filename"
        :content-type="contentType"
        :prefix="prefix"
        w-full
      />
    </div>
    <div
      v-else-if="inputMode === 'text'"
      flex="~ col" h-full w-full gap-3
    >
      <textarea
        v-model="textInput"
        placeholder="Enter text to transmit..."
        border="~ gray/25 rounded-lg"
        w-full flex-1 resize-none bg-transparent p-4 text-sm shadow-sm
      />
      <button
        w-full btn
        :disabled="!textInput.trim()"
        @click="onTextSubmit"
      >
        Start Transmission
      </button>
    </div>
    <template v-else>
      <InputFile
        text="neutral-600 dark:neutral-400"
        aspect-1 h-full w-full rounded-lg
        @file="onFileChange"
      />
      <DropZone text="Drop File Here" @file="onFileChange" />
    </template>
  </div>
</template>
