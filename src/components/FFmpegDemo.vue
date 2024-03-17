<script setup lang="ts">
import { FFmpeg } from '@ffmpeg/ffmpeg'
import type { LogEvent } from '@ffmpeg/ffmpeg/dist/esm/types'
import { fetchFile, toBlobURL } from '@ffmpeg/util'
import { ref } from 'vue'
import { saveAs } from 'file-saver'
import data from '../../public/test.json'

const baseURL = 'https://unpkg.com/@ffmpeg/core-mt@0.12.6/dist/esm'

const ffmpeg = new FFmpeg()
const message = ref('Click Start to Transcode')
let video = ref('')

ffmpeg.on('log', ({ message: msg }: LogEvent) => {
  message.value = msg
  console.log(msg)
})

await ffmpeg.load({
  coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
  wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm'),
  workerURL: await toBlobURL(`${baseURL}/ffmpeg-core.worker.js`, 'text/javascript')
})

const OUT_FILE_NAME = 'output.mp4'
const FINAL_OUTPUT_FILE = 'final.mkv'
const FONT_FILE = 'Roboto-Regular.ttf'
const AUDIO_FILE = 'audio.wav'

// Fetch and write the font file for drawtext filter
await ffmpeg.writeFile(FONT_FILE, await fetchFile('/Roboto-Regular.ttf'))

// Fetch and write the audio file
await ffmpeg.writeFile(AUDIO_FILE, await fetchFile('/audio.wav'))

type WordInfo = {
  word: string
  start: number
  end: number
}

const generateFfmpegCommands = (data: { word: string; start: number; end: number }[]) => {
  const baseFfmpegCmd = ['-i', OUT_FILE_NAME, '-i', AUDIO_FILE]
  const filterComplexCmd: string[] = []
  const maxChars = 15 // Limit by character count

  // Helper function to chunk data by character count
  const chunkDataByCharCount = (
    data: { word: string; start: number; end: number }[],
    maxChars: number
  ) => {
    const chunks: { text: string; start: number; end: number }[] = []
    let currentChunk = ''
    let chunkStart = data[0]?.start || 0
    let chunkEnd = 0

    data.forEach((item, index) => {
      if (currentChunk.length + item.word.length + (currentChunk.length > 0 ? 1 : 0) <= maxChars) {
        // Add a space before the word if it's not the first word in the chunk
        currentChunk += (currentChunk.length > 0 ? ' ' : '') + item.word
        chunkEnd = item.end
      } else {
        // Save the current chunk and reset for the next chunk
        chunks.push({ text: currentChunk, start: chunkStart, end: chunkEnd })
        currentChunk = item.word
        chunkStart = item.start
        chunkEnd = item.end
      }

      // Ensure the last chunk is added
      if (index === data.length - 1) {
        chunks.push({ text: currentChunk, start: chunkStart, end: chunkEnd })
      }
    })

    return chunks
  }

  const textChunks = chunkDataByCharCount(data, maxChars)

  // Generate drawtext commands for each chunk
  textChunks.forEach((chunk) => {
    const formattedWord = chunk.text.replace("'", '')
    const drawtextCmd = `drawtext=fontfile=Roboto-Regular.ttf:text='${formattedWord}':enable='between(t,${chunk.start},${chunk.end})':x=(w-tw)/2:y=7*(h-th)/8:fontsize=48:fontcolor=white:box=1:boxcolor=black@0.5:boxborderw=5`
    filterComplexCmd.push(drawtextCmd)
  })

  const complexFilterStr = filterComplexCmd.join(',')

  const finalCmd = [
    ...baseFfmpegCmd,
    '-filter_complex',
    complexFilterStr,
    '-map',
    '0:v', // Map video stream(s) from the first input
    '-map',
    '1:a', // Map audio stream(s) from the second input
    '-c:a',
    'copy', // Copy audio codec, assuming no filters are applied to audio
    '-shortest', // Finish encoding when the shortest stream ends
    FINAL_OUTPUT_FILE
  ]

  console.log(finalCmd)

  return finalCmd
}

const generateCreateVideoCommands = (data: { word: string; start: number; end: number }[]) => {
  const videoLength = data[data.length - 1].end // Duration of the video
  const videoDimensions = '540x960'
  const ffmpegCommandArray = [
    '-f',
    'lavfi',
    '-i',
    `color=c=white:s=${videoDimensions}:d=${videoLength}`,
    '-c:v',
    'libx264',
    '-t',
    `${videoLength}`,
    '-pix_fmt',
    'yuv420p',
    OUT_FILE_NAME
  ]

  return ffmpegCommandArray
}

async function generateVideo() {
  const videoCmd = generateCreateVideoCommands(data.word_segments)
  await ffmpeg.exec(videoCmd)

  const ffmpegCommand = await generateFfmpegCommands(data.word_segments)
  await ffmpeg.exec(ffmpegCommand)

  const out_file = await ffmpeg.readFile(FINAL_OUTPUT_FILE)
  const blob = new Blob([(out_file as Uint8Array).buffer], { type: 'video/mp4' })
  saveAs(blob, FINAL_OUTPUT_FILE)
  video.value = URL.createObjectURL(
    new Blob([(out_file as Uint8Array).buffer], { type: 'video/mp4' })
  )
}
</script>

<template>
  <div class="video-container">
    <video :src="video" controls />
  </div>
  <br />
  <button @click="generateVideo">Start</button>
  <p>{{ message }}</p>
</template>

<style scoped>
.video-container {
}
</style>
