<template>
  <div class="preview">
    <!-- 无法预览，显示下载与直链 -->
    <a-result :title="file.name" v-if="previewShow.other">
      <template #icon>
        <component :is="file.icon" class="file-icon" />
      </template>
      <template #extra>
        <a target="_blank" :href="downloadUrl">
          <a-button type="primary">下载</a-button>
        </a>
        <a-button type="primary" @click="copyFileLink">复制直链</a-button>
      </template>
    </a-result>
    <!-- 显示加载的部分 -->
    <a-spin :spinning="previewSpinning" v-if="previewShow.spinning">
      <!-- 文档预览 -->
      <div class="doc-preview" id="doc-preview" v-if="previewShow.doc"></div>
      <!-- iframe -->
      <iframe
        :src="downloadUrl"
        id="iframe-preview"
        ref="iframe-preview"
        v-if="previewShow.iframe"
        allowfullscreen="allowfullscreen"
        webkitallowfullscreen="true"
        mozallowfullscreen="true"
        class="iframe-preview"
        frameborder="no"
        @load="previewSpinning = false"
      ></iframe>
      <!-- 图片预览 -->
      <div class="img-preview" v-if="previewShow.image">
        <img @load="previewSpinning = false" :src="downloadUrl" />
      </div>
      <!-- 文本预览 -->
      <div class="text-preview" v-if="previewShow.text">
        <v-md-preview :text="text"></v-md-preview>
      </div>
    </a-spin>
    <!-- 视频预览 -->
    <div v-show="previewShow.video">
      <a-switch checked-children="转码" un-checked-children="原画" v-model:checked="videoTranscoding" />
      <div
        class="video-preview"
        id="video-preview"
      ></div>
    </div>
    
    <!-- 音频预览 -->
    <div class="audio-preview" v-show="previewShow.audio" id="audio-preview"></div>
  </div>
</template>

<script lang="ts">
import useDownloadUrl from "@/hooks/useDownloadUrl";
import { FileProps, GlobalDataProps } from "@/store";
import { getPost, getText, officePreviewPost, videoPreviewPost } from "@/utils/api";
import { doc } from "@/utils/const";
import { getIcon } from "@/utils/get_icon";
import { message } from "ant-design-vue";
import { computed, defineComponent, onBeforeUnmount, onMounted, ref, watch } from "vue";
import { useStore } from "vuex";
import DPlayer, { DPlayerOptions } from 'dplayer'
import { useRoute } from "vue-router";
import 'aplayer/dist/APlayer.min.css';
import APlayer from 'aplayer';

declare namespace aliyun {
  class Config {
    setToken(token: { token: string });
  }
  function config({mount: Element, url: string}): Config;
}

interface PreviewShow {
  image: boolean;
  video: boolean;
  audio: boolean;
  doc: boolean;
  other: boolean;
  text: boolean;
  iframe: boolean;
  spinning: boolean;
}

export default defineComponent({
  name: "Preview",
  setup() {
    const store = useStore<GlobalDataProps>()
    const route = useRoute()
    const file = computed<FileProps>(() => {
      const res = store.state.data as FileProps
      res.icon = getIcon(res)
      return res
    });
    const {downloadUrl, copyFileLink} = useDownloadUrl()
    const info = computed(() => store.state.info)
    const previewSpinning = ref<boolean>(true)
    const previewShow = ref<PreviewShow>({
      image: false,
      video: false,
      audio: false,
      doc: false,
      other: false,
      text: false,
      iframe: false,
      spinning: false,
    });
    let dp,ap
    const audios = computed(() => store.state.audios)
    const text = ref<string>('')
    const showDoc = (file: FileProps) => {
      previewShow.value.spinning = true
      previewSpinning.value = true
      previewShow.value.doc = true
      officePreviewPost(store.state.drive, file.file_id).then((resp) => {
        const res = resp.data
        if (res.meta.code === 200) {
          const docOptions = aliyun.config({
            mount: document.querySelector("#doc-preview"),
            url: res.data.preview_url, //设置文档预览URL地址。
          })
          docOptions.setToken({ token: res.data.access_token })
          setTimeout(() => {
            previewSpinning.value = false
          }, 200)
        } else {
          message.error(res.meta.msg)
        }
      });
    };
    const showFile = async (file: FileProps) => {
      if (doc.includes(file.file_extension.toLowerCase())) {
        showDoc(file)
        return
      }
      if (file.category=='image') {
        // 预览图片
        previewShow.value.spinning = true
        previewSpinning.value = true
        previewShow.value.image = true
        return
      }
      if (file.category==='video') {
        // 预览视频
        const {data} = await getPost(decodeURI(route.path.substring(1)),store.state.password)
        previewShow.value.video=true
        const ex = file.file_extension
        let type = 'auto'
        if(ex==='flv'){
          type = 'flv'
        }
        // if(ex==='ts'){
        //   type = 'hls'
        // }
        const videoOptions: DPlayerOptions={
          container: document.getElementById('video-preview'),
          video:{
            url: data.data.url,
            type: type
          },
          pluginOptions: {
            flv: {
              config: {
                referrerPolicy: 'no-referrer'
              }
            }
          },
          autoplay:info.value.autoplay?true:false,
          screenshot:true,
        }
        dp=new DPlayer(videoOptions)
        return
      }
      if (file.category ==='audio') {
        previewShow.value.audio=true
        const audioOptions = {
          container: document.getElementById('audio-preview'),
          autoplay: true,
          audio: [{
            name: file.name,
            url: downloadUrl.value,
            cover: info.value.music_img,
          }, ...audios.value],
          listMaxHeight: '65vh'
        }
        ap = new APlayer(audioOptions)
        return
      }
      if(info.value.preview?.text.includes(file.file_extension.toLowerCase())){
        previewShow.value.text = true
        previewSpinning.value = true
        previewShow.value.spinning = true
        getPost(file.dir+file.name, store.state.password).then(resp=>{
          const res = resp.data
          if(res.meta.code===200){
            getText(res.data.url).then(resp=>{
              if(file.file_extension.toLowerCase()==='md'){
                text.value = resp.data
              }else if(file.file_extension.toLowerCase()==='json'){
                text.value = '```'+file.file_extension.toLowerCase()+'\n'+JSON.stringify(resp.data, null, 2)+'\n```'
              }else{
                text.value = '```'+file.file_extension.toLowerCase()+'\n'+resp.data+'\n```'
              }
              previewSpinning.value = false
            })
          }else{
            message.error(res.meta.msg)
          }
        })
        return
      }
      previewShow.value.other = true
    };
    onMounted(() => {
      showFile(file.value);
    });
    onBeforeUnmount(() => {
      if(ap){
        ap.destroy()
      }
      if(dp){
        dp.destroy()
      }
    })
    const videoTranscoding = ref<boolean>(false);
    watch(videoTranscoding, async () => {
      if(dp){
        dp.destroy()
      }
      if(!videoTranscoding.value) {
        const {data} = await getPost(decodeURI(route.path.substring(1)),store.state.password)
        const ex = file.value.file_extension
        let type = 'auto'
        if(ex==='flv'){
          type = 'flv'
        }
        // if(ex==='ts'){
        //   type = 'hls'
        // }
        const videoOptions: DPlayerOptions={
          container: document.getElementById('video-preview'),
          video:{
            url: data.data.url,
            type: type
          },
          pluginOptions: {
            flv: {
              config: {
                referrerPolicy: 'no-referrer'
              }
            }
          },
          autoplay:info.value.autoplay?true:false,
          screenshot:true,
        }
        dp=new DPlayer(videoOptions)
      }else{
        videoPreviewPost(store.state.drive, file.value.file_id).then(resp => {
          const res = resp.data
          if (res.meta.code === 200) {
            const videoOptions: DPlayerOptions={
            container: document.getElementById('video-preview'),
            video:{
              url:'',
              quality: res.data.template_list.map(item => {
                return{
                  name: item.template_id,
                  url: item.url,
                  type: 'auto'
                }
              }),
              defaultQuality: 0,
            },
            pluginOptions: {
              flv: {
                config: {
                  referrerPolicy: 'no-referrer'
                }
              }
            },
            autoplay:info.value.autoplay?true:false,
            // screenshot:true,
          }
          dp=new DPlayer(videoOptions)
          } else {
            message.error(res.meta.msg)
          }
        })
      }
    })
    return {
      file,
      previewSpinning,
      previewShow,
      downloadUrl,
      copyFileLink,
      text,
      videoTranscoding,
    };
  },
});
</script>

<style scoped>
.video-preview {
  width: 100%;
  margin-top: 5px;
}
/* @media screen and (min-width: 600px) {
  #video-preview{
    height: 80vh;
  }
} */
.iframe-preview {
  width: 100%;
  height: 80vh;
  box-sizing: inherit;
}
.doc-preview {
  width: 100%;
  height: 80vh;
}
.img-preview {
  width: 100%;
  height: 80vh;
}
.img-preview img {
  max-width: 100%;
  max-height: 100%;
  display: block;
  margin: auto;
}
</style>
