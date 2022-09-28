<script lang="ts" setup>
import { computed, onMounted, ref } from 'vue';
import { Greet } from '../wailsjs/go/main/App'
const server = ref("http://localhost:8080")
const serverHost = computed(() => {
  const result = /^https?\:\/\/(.+)\:/.exec(server.value)
  if (result != null) {
    return result[1]
  }
  return server.value
})
interface Stream {
  Path: string, Tracks: string[], Type: string
}
const streamList = ref<Stream[]>([])
let plugins: Record<string, { Name: string, RawConfig: Record<string, any> }> | null = null
function fetchPlugins() {
  fetch(server.value + "/api/plugins").then(x => x.json()).then(x => {
    plugins = x
  })
}
onMounted(() => {
  setInterval(() => {
    fetch(server.value + "/api/summary").then(x => x.json()).then(x => {
      streamList.value = x.Streams.sort((a: Stream, b: Stream) => a.Path > b.Path ? 1 : -1)
      if (!plugins) {
        fetchPlugins()
      }
    }).catch(() => {
      streamList.value = []
    })
  }, 1000)
})
const playError = ref("")
function play(url: string) {
  Greet(url).then(err => {
    playError.value = err
  })
}
function getPlayUrls(s: Stream) {
  const result: string[] = [

  ]
  if (plugins) {
    if (plugins['RTMP'])
      result.push(`rtmp://${serverHost.value}${plugins['RTMP'].RawConfig.tcp.listenaddr}/${s.Path}`)
    if (plugins['HLS'])
      result.push(`${server.value}/hls/${s.Path}.m3u8`)
    if (plugins['HDL'])
      result.push(`${server.value}/hdl/${s.Path}.flv`)
    if (plugins['RTSP'])
      result.push(`rtsp://${serverHost.value}/${s.Path}`)
  }
  return result
}
</script>

<template>
  <n-layout>
    <n-layout-header>
      <n-input-group>
        <n-input-group-label>m7s的http服务地址</n-input-group-label>
        <n-input v-model:value="server" placeholder="http://localhost:8080" />
      </n-input-group>
    </n-layout-header>
    <n-layout-content content-style="padding: 24px;">
      <n-collapse>
        <n-collapse-item :title="s.Path" :name="s.Path" :key="s.Path" v-for="s in streamList">
          <template v-for="url in getPlayUrls(s)">
            <a href="javascript:void(0)" @click="play(url)">{{url}}</a><br>
          </template>
          <template #header-extra>
            <n-tag> {{s.Type}} </n-tag>
            <n-tag v-for="t in s.Tracks"> {{t}} </n-tag>
          </template>
        </n-collapse-item>
      </n-collapse>
    </n-layout-content>
    <n-layout-footer>{{playError}}</n-layout-footer>
  </n-layout>
</template>

<style>

</style>
