<script setup lang="ts">
import { FFmpeg } from '@ffmpeg/ffmpeg'
import type { LogEvent } from '@ffmpeg/ffmpeg/dist/esm/types'
import { fetchFile, toBlobURL } from '@ffmpeg/util'
import { saveAs } from 'file-saver'

import responseJson from '../../public/response000.json'
import transcriptData from '../../public/transcript000.json'

interface ClipMetaData {
  start: number
  end: number
  text: string
}

interface ChatGPTResponse {
  data: ClipMetaData[]
}

interface TranscriptResponse {
  data: TranscriptData[]
}

interface TranscriptData {
  text: string
  timestamp: number[]
}

const baseURL = 'https://unpkg.com/@ffmpeg/core-mt@0.12.6/dist/esm'
const ffmpeg = new FFmpeg()
const cutLength = {
  start: responseJson.data[1].start,
  end: responseJson.data[1].end
}

ffmpeg.on('log', ({ message: msg, type }: LogEvent) => {
  if (type === 'error') {
    console.error(msg)
  } else {
    console.log(msg)
  }
})

await ffmpeg.load({
  coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
  wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm'),
  workerURL: await toBlobURL(`${baseURL}/ffmpeg-core.worker.js`, 'text/javascript')
})

const FINAL_OUTPUT_FILE = 'final.mp4'

const RECORDING_FILE = 'output000.mp4'
await ffmpeg.writeFile(RECORDING_FILE, await fetchFile('/output000.mp4'))

// Fetch and write the font file for drawtext filter
const FONT_FILE = 'Roboto-Regular.ttf'
await ffmpeg.writeFile(FONT_FILE, await fetchFile('/Roboto-Regular.ttf'))

const generateFfmpegCommands = (transcriptData: TranscriptData[], recordingFilePath: string) => {
  const baseFfmpegCmd = ['-i', recordingFilePath]
  const filterComplexCmd: string[] = []
  const maxChars = 15 // Limit by character count

  const extractSegments = (
    transcript: TranscriptData[],
    start: number,
    end: number
  ): TranscriptData[] => {
    return transcript.filter((segment) => {
      // Check if the segment's start time is within the given range.
      return segment.timestamp[0] >= start && segment.timestamp[0] <= end
    })
  }

  // Helper function to chunk data by character count
  const chunkDataByCharCount = (transcriptData: TranscriptData[], maxChars: number) => {
    const chunks: ClipMetaData[] = []
    let currentChunk = ''
    let chunkStart = cutLength.start
    let chunkEnd = cutLength.end

    const transcript = extractSegments(transcriptData, cutLength.start, cutLength.end)
    console.log('transcriptData: ', transcriptData)
    console.log('transcript: ', transcript)

    transcript.forEach((item, index) => {
      if (currentChunk.length + item.text.length + (currentChunk.length > 0 ? 1 : 0) <= maxChars) {
        // Add a space before the word if it's not the first word in the chunk
        currentChunk += (currentChunk.length > 0 ? ' ' : '') + item.text
        chunkEnd = item.timestamp[1]
      } else {
        // Save the current chunk and reset for the next chunk
        chunks.push({ text: currentChunk, start: chunkStart, end: chunkEnd })
        currentChunk = item.text
        chunkStart = item.timestamp[0]
        chunkEnd = item.timestamp[1]
      }

      // Ensure the last chunk is added
      if (index === transcriptData.length - 1) {
        chunks.push({ text: currentChunk, start: chunkStart, end: chunkEnd })
      }
    })

    return chunks
  }

  const textChunks = chunkDataByCharCount(transcriptData, maxChars)

  // Generate drawtext commands for each chunk
  textChunks.forEach((chunk) => {
    const formattedWord = chunk.text.replace("'", '')

    const drawtextCmd = `drawtext=fontfile=Roboto-Regular.ttf:text='${formattedWord}':enable='between(t,${chunk.start - cutLength.start},${chunk.start - cutLength.start + chunk.end - chunk.start})':x=(w-tw)/2:y=(h-th)/2:fontsize=96:fontcolor=white:box=1:boxcolor=black@0.8:boxborderw=4`
    // const drawtextCmd = `drawtext=fontfile=Roboto-Regular.ttf:text='${formattedWord}':enable='between(t,${chunk.start},${chunk.end})':x=(w-tw)/2:y=7*(h-th)/8:fontsize=48:fontcolor=white:box=1:boxcolor=black@0.5:boxborderw=5`

    filterComplexCmd.push(drawtextCmd)
  })

  const complexFilterStr = filterComplexCmd.join(',')

  // const finalCmd = [...baseFfmpegCmd, '-filter_complex', complexFilterStr, FINAL_OUTPUT_FILE]
  const finalCmd = [
    ...baseFfmpegCmd,
    '-filter_complex',
    complexFilterStr,
    '-c:v',
    'libx264',
    '-crf',
    '23',
    '-c:a',
    'copy',
    FINAL_OUTPUT_FILE
  ]

  console.log(finalCmd)

  return finalCmd
}

async function generateVideo() {
  const filePath = await cutRecordingsFile(RECORDING_FILE, cutLength.start, cutLength.end)
  await fetchFile(filePath)

  // TODO: This is not saving the segmented files
  // await chopIntoSegments(RECORDING_FILE)

  const ffmpegCommand = await generateFfmpegCommands(transcriptData, filePath)
  console.log('pre: ffmpegCommand: ', ffmpegCommand)
  await ffmpeg.exec(ffmpegCommand)

  console.log('post: ffmpegCommand: ', ffmpegCommand)

  const out_file = await ffmpeg.readFile(FINAL_OUTPUT_FILE)
  const blob = new Blob([(out_file as Uint8Array).buffer], { type: 'video/mp4' })
  saveAs(blob, FINAL_OUTPUT_FILE)

  console.log('saved final output')
}

const cutRecordingsFile = async (filePath: string, start: number, end: number) => {
  // ffmpeg -i output000.mp4 -ss 00:02:15 -to 00:04:45 cut-output000.mp4
  const newFilePath = 'cut-' + filePath
  const args = [
    '-ss',
    start.toString(),
    '-to',
    end.toString(),
    '-i',
    filePath,
    '-c',
    'copy',
    newFilePath
  ]
  await ffmpeg.exec(args)

  // const out_file = await ffmpeg.readFile(newFilePath)

  // const fileType = filePath.split('.').pop()
  // const mimeType = `video/${fileType}`

  // const blob = new Blob([(out_file as Uint8Array).buffer], { type: mimeType })
  // saveAs(blob, newFilePath)

  return newFilePath
}

// await cutRecordingTest()

const chopIntoSegments = async (filePath: string) => {
  // ffmpeg -i input.mp4 -c copy -f segment -segment_time 200 -reset_timestamps 1
  const args = [
    '-i',
    filePath,
    '-c',
    'copy',
    '-f',
    'segment',
    '-segment_time',
    '200',
    'output%03d.mp4'
  ]

  await ffmpeg.exec(args)

  const newFilePath = 'output002.mp4'

  const fileType = filePath.split('.')[1]
  const out_file = await ffmpeg.readFile(newFilePath)
  const blob = new Blob([(out_file as Uint8Array).buffer], { type: fileType })
  saveAs(blob, newFilePath)

  await ffmpeg.writeFile(newFilePath, await fetchFile(newFilePath))

  // return newFilePath
}
</script>

<template>
  <button @click="generateVideo">Start</button>
  <!-- <button
    @click="() => cutFileLength(AUDIO_FILE, cutLength.start.toString(), cutLength.end.toString())"
  >
    Cut Audio File
  </button>
  <button
    @click="
      () => cutFileLength(RECORDING_FILE, cutLength.start.toString(), cutLength.end.toString())
    "
  >
    Cut Recording File
  </button> -->
</template>
