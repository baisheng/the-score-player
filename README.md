![用VUE实现一个乐谱播放器](https://upload-images.jianshu.io/upload_images/1511070-e86fba3c8ecd2b89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 前言：
整理一个年初做的一个APP项目，项目中的核心功能是与【一起练琴】这个 app 较为相似，其中最重点的是乐谱播放和根据乐谱练习评分的功能。今天我们主要介绍下我们选用的一个乐谱播放的方案。

## 重点调研
经过大量的调研在乐谱可视化方向已经发展多年，有不少成熟的开源解决方案，层层筛选后有三个库比较适合我们的需求（支持乐谱定制、音源定制、musicxml 格式支持）：

[alphaTab – music notation for everyone](https://www.alphatab.net/) 这个是基于 typescript 的实现，主攻吉他谱，当然其他乐谱也支持。
**特点：可以识别乐谱和定制音源播放，可以读取 musicxml格式**
![image.png](https://upload-images.jianshu.io/upload_images/1511070-a993196858d9e380.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)


[VexFlow - HTML5 Music Engraving](http://www.vexflow.com/) 一个用于渲染音乐符号的JavaScript库，音源播放需要定制开发。
![image.png](https://upload-images.jianshu.io/upload_images/1511070-7b97624fea5d93ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)


[opensheetmusicdisplay](https://opensheetmusicdisplay.github.io/demo/) 一个开源的乐谱播放器。
**特点：简单，支持 musicxml 乐谱文件，支持音源，支持 svg、canvas 两种渲染模式**
![image.png](https://upload-images.jianshu.io/upload_images/1511070-d36c2d513b8246e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)



## 选型
基于以上三个开源库进行定制开发，发现会有不同的问题：
**AlphaTap**
缺点：musicxml支持不完善，样式定制较为困难，音源定制不灵活
优点：美观、屏幕自适应完善、定制开发容易

**VexFlow**
缺点：这个应该也不算是缺点，因为它是一个乐谱绘制库，所以对于musicxml、音源等支持，需要用第三方基于它实现的库，或者我们自己开发，但是当时的项目周期太紧张，最好还是放弃掉了。
优点：强大、与原生 app 结合好、有基于 flutter 的方案

**OpenSheetMusicDisplay**
缺点：简陋，需要优化的地方较多，音源需要定制
优点：这个是完整的乐谱播放器实现，基础的乐谱播放功能、音源加载、乐谱格式 musicxml 支持都比较完整。

**最终我们选择提 OSMD 这个方案，下面讲一下我们是如何使用的**

## 开发
为了兼容移动展示，和最终嵌入到原生 app 中，我们使用了几个库：
```
$ yarn add vue opensheetmusicdisplay vant vue-webview-js-bridge 

vant ui # 适配移动端组件的 UI
vue-webview-js-bridge # 与原生 app 通信
```
**项目结构如下**
```
 .
├── LICENSE
├── README.md
├── babel.config.js
├── package-lock.json
├── package.json
├── postcss.config.js
├── public
│   ├── Happy_Birthday.musicxml
│   ├── static
│   └── voice.svg
├── src
│   ├── App.vue
│   ├── assets
│   ├── components
│   ├── layout
│   ├── main.js
│   ├── plugins
│   ├── router
│   ├── scores.js
│   └── views
├── vue.config.js
├── yarn-error.log
└── yarn.lock

```
可以看出非常简单，就是一个基础的 vue 工程，主要是 view 中的 score.vue，它整合了 OSMD 播放器，和加载乐谱的方式。看下重点代码：
**views/score.vue**
```
<template>
  <div>
    <van-nav-bar
            right-text="配置"
            @click-left="onClickLeft"
            @click-right="onClickRight"
            fixed
            :border="false"
            style="background: transparent"
    >
      <template #right>
    <!-- 节拍器 -->
        <metronome-view :intf="intf" />
        <metronome
                ref="metronome"
                :gain="gain"
                :tempo="tempo"
                :beat="beat"
                :mute="mute"
                :pause="pause"
                @start="onMetronomeStart"
                @stop="onMetronomeStop"
        />
      </template>
    </van-nav-bar>

    <div class="music-score">
      <Score
              v-if="mounted"
              @osmdInit="osmdInit"
              @scoreLoaded="scoreLoaded"
              :score="selectedScore"
              :ready="pbEngineReady"
      />
...
</template>

<script>
  import { Tabbar, TabbarItem } from 'vant';
  import { Button } from 'vant';
  import {MetronomeView, Metronome} from '../components/metronome'
  import PlayButton from "../components/PlayButton";
  import PlaybackSidebar from "../components/PlaybackSidebar";
  import Score from "../components/Score";
  import scores from "../scores";

  import PlaybackEngine from "../../../../dist/PlaybackEngine";
  import { EventBus } from '../plugins/event-bus.js';

  const PlaybackState = {
    INIT: "INIT",
    PLAYING: "PLAYING",
    STOPPED: "STOPPED",
    PAUSED: "PAUSED",
    FINISHED: "FINISHED"
  }

  export default {
    name: "app",
    components: {
      PlayButton,
      Metronome,
      MetronomeView,
      osmd: null,
      Score,
      VanButton: Button,
      VanTabbar: Tabbar,
      VanTabbarItem: TabbarItem,
    },
    data() {
      return {
        sheetScore: '',
        active: 0,
        pbEngine: new PlaybackEngine(),
        pbEngineReady: false,
        scores: scores,
        selectedScore: null,
        osmd: null,
        scoreTitle: "",
        drawer: true,
        mounted: false,
        onMusicMode: true,
        onMetronome: false,
        onGameMode: false,

        metronomeShow: false,
        beat: 4,
        gain: -30,
        tempo: 120,
        mute: false,
        pause: false,
        playing: false,
        intf: null
      };
    },
 
    methods: {
      initBridge() { 
        ...
      },
      osmdInit(osmd) {
        this.osmd = osmd;
        this.selectedScore = this.sheetScore
      },
      async scoreLoaded() {
        // console.log("Score loaded");
        if (this.osmd.sheet.title) this.scoreTitle = this.osmd.sheet.title.text;
        await this.pbEngine.loadScore(this.osmd);
        this.pbEngineReady = true;
        // 加载完成
        await this.callNativeFinishPlay()
      },
      scoreChanged(scoreUrl) {
        if (this.pbEngine.state === "PLAYING") this.pbEngine.stop();
        this.selectedScore = scoreUrl;
        this.pbEngineReady = false;
      },
    },
    computed: {
      isGamePlaying() {
        return this.onGameMode && this.playing
      },
      isMusicPlaying() {
        return !this.onGameMode && this.playing
      }
    },
    mounted() {
      const query = this.$route.query
      if (query.score) {
        this.sheetScore = query.score
      }
      // 停止时调用原生触发
      EventBus.$on(PlaybackState.STOPPED, () => {
        // this.callNativeFinishPlay()
        this.callNativeEndPlay()
      })
      this.initBridge()
      setTimeout(() => {
        // This extra delay before rendering the score component seems to help occasional issues where the
        // OSMD cursor img element gets detached from the DOM and doesn't show unless
        // you refresh the page. A less pretty workaround until root cause is determined
        this.mounted = true;
      }, 200)
    }
  };
</script>

```
**components/Score.vue**

```
<script>
import axios from "axios";
import { OpenSheetMusicDisplay } from "opensheetmusicdisplay";

export default {
  props: ["score", "ready"],
  data() {
    return {
      osmd: null,
      scoreLoading: false
    };
  },
  watch: {
    score(val, oldVal) {
      if (!val || val === oldVal) return;
      this.loadScore(val);
    },
    ready(val, oldVal) {
      if (!val || val === oldVal) return
      this.$toast.clear()
    }
  },
  async mounted() {
    this.osmd = new OpenSheetMusicDisplay(this.$refs.scorediv, {
      followCursor: true,
      autoResize: false,
    });
    this.$emit("osmdInit", this.osmd);
    if (this.score) this.loadScore(this.score);
    this.osmd.rules.DefaultColorCursor = '#008be8'
  },
  methods: {
    async loadScore(scoreUrl) {
      this.scoreLoading = true;
      // this.osmd.rules.DefaultColorCursor = '#008be8'

      let scoreXml = await axios.get(scoreUrl);
      await this.osmd.load(scoreXml.data);

      this.osmd.rules.DefaultColorCursor = '#008be8'

      this.scoreLoading = false;
      await this.$nextTick();
      await this.osmd.render();
      if (this.scoreLoading || !this.ready) {

        // this.$toast.loading({
        //   message: '加载中...',
        //   forbidClick: true,
        //   loadingType: 'spinner',
        // });
      }
      this.$emit("scoreLoaded");
    }
  }
};
</script>
```

**以上两个文件的主要作用是，**
- score component 用来引入和整合 osmd 模块，包含加载乐谱、加载音频、乐谱渲染
- score view 作为入口页面用来访问乐谱，桥接与原生 app 通信控制
  **最终效果：**
  ![image.png](https://upload-images.jianshu.io/upload_images/1511070-1e17025d7a6dcf6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**做个样机展示下：**
![Hello.png](https://upload-images.jianshu.io/upload_images/1511070-0b3f2e3faad6bf00.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结
本篇文章主要介绍了乐谱的可视化方案，并将方案应用在实际项目中，并做出示例。可以给有相关需求的朋友做为参考。

## 相关资源
https://github.com/baisheng/osmd-player **文章中的系统源码**

[alphaTab – music notation for everyone](https://www.alphatab.net/)
[VexFlow - HTML5 Music Engraving](http://www.vexflow.com/)
[opensheetmusicdisplay](https://opensheetmusicdisplay.github.io/demo/)


# 🎵 我们的合奏 OSMD Audio player

## Build

```
yarn build
```

## Apps

### The Score
我们的合奏 h5 端

**Live demo:** https://test.picker.cc/ <br/>

```
yarn install
yarn serve
yarn build
```

### 原生调用
原生调H5，都不需要参数
1，开始播放  startPlay
2，暂停播放  pausePlay
3，结束播放  endPlay
H5调原生
1，切换模式  changeModel  需要参数“model”  1是欣赏模式；2是评分模式
2，停止播放  endPlay  不需要参数

### 改光标颜色问题
等 0.9.3

### ISSUE
**Hide measures that includes invisible notes.**
https://github.com/opensheetmusicdisplay/opensheetmusicdisplay/issues/913

### 一些资源
https://github.com/opensheetmusicdisplay/opensheetmusicdisplay/issues/393
**关于 cursor 换颜色的问题，要等0.9.3 版本**
https://github.com/opensheetmusicdisplay/opensheetmusicdisplay/discussions/961
**osmd-options**
https://github.com/opensheetmusicdisplay/opensheetmusicdisplay/wiki/Getting-Started#osmd-options
https://github.com/opensheetmusicdisplay/opensheetmusicdisplay/blob/1c49122b4bf2c38380e7c291451e02ccdffa0b2c/src/OpenSheetMusicDisplay/OSMDOptions.ts

# 🎵 我们的合奏 OSMD Audio player

## Build

```
yarn build
```

## Apps

### The Score
我们的合奏 h5 端

**Live demo:** https://test.picker.cc/ <br/>

```
yarn install
yarn serve
yarn build
```

### 原生调用
原生调H5，都不需要参数
1，开始播放  startPlay
2，暂停播放  pausePlay
3，结束播放  endPlay
H5调原生
1，切换模式  changeModel  需要参数“model”  1是欣赏模式；2是评分模式
2，停止播放  endPlay  不需要参数

### 改光标颜色问题
等 0.9.3

### ISSUE
**Hide measures that includes invisible notes.**
https://github.com/opensheetmusicdisplay/opensheetmusicdisplay/issues/913

### 一些资源
https://github.com/opensheetmusicdisplay/opensheetmusicdisplay/issues/393
**关于 cursor 换颜色的问题，要等0.9.3 版本**
https://github.com/opensheetmusicdisplay/opensheetmusicdisplay/discussions/961
**osmd-options**
https://github.com/opensheetmusicdisplay/opensheetmusicdisplay/wiki/Getting-Started#osmd-options
https://github.com/opensheetmusicdisplay/opensheetmusicdisplay/blob/1c49122b4bf2c38380e7c291451e02ccdffa0b2c/src/OpenSheetMusicDisplay/OSMDOptions.ts
