<template>
  <div class="player" v-show="playlist.length>0">
     <transition name="normal" > 
      <div class="normal-player" v-show="fullScreen">
          <div class="background">
              <div class="filter"></div>
              <img :src="currentSong.image" width="100%" height="100%"/>
          </div>
          <div class="top">
              <div class="back" @click="back" >
                  <font-awesome-icon icon="angle-down" class="icon"></font-awesome-icon>
              </div>
              <h1 class="title" v-html="currentSong.name"></h1>
              <h2 class="subtitle" v-html="currentSong.singer"></h2>
          </div>
          <div class="middle" @click="changeMiddle">
              <transition name="middleL">
                   <div class="middle-l" v-show="currentShow === 'cd'">
                        <div class="cd-wrapper">
                            <div class="cd" :class="cdClass" >
                                <img class="image" :src="currentSong.image">
                            </div>
                        </div>
                        <div class="playing-lyric-wrapper">
                            <div class="playing-lyric">
                                {{playingLyric}}
                            </div>
                        </div>
                    </div>
              </transition>
              <transition name="middleR">
              <scroll class="middle-r" ref="lyricList" v-show="currentShow === 'lyric'"  :data="currentLyric && currentLyric.lines">
                  <div class="lyric-wrapper">
                      <div class="currentLyric" v-if="currentLyric">
                          <p ref="lyricline"
                             class="text"
                             :class="{'current':currentLineNum === index}"
                             v-for="(line,index) in currentLyric.lines" :key="index">
                            {{line.txt}}
                          </p>
                      </div>
                      <p class="no-lyric"  v-if="currentLyric === null">
                          {{upDatecurrentLyric}}
                      </p>
                  </div>
              </scroll>
              </transition>
          </div>
          <div class="bottom">
              <div class="dot-wrapper">
                  <span class="dot" :class="{'active':currentShow==='cd'}"></span>
                  <span class="dot" :class="{'active':currentShow==='lyric'}"></span>
                </div>
              <div class="progress-wrapper">
                  <span class="time time-l">{{format(currentTime)}}</span>
                  <div class="progress-bar-wrapper">
                    <progress-bar @percentChange="onProgressBarChange" :percent="percent"></progress-bar>
                  </div>
                  <span class="time time-r">{{format(duration)}}</span>
              </div>
              <div class="operators">
                    <div class="icon i-left" >
                    <!-- <i class="iconfont mode" :class="iconMode" @click="changeMode"></i> -->
                        <font-awesome-icon :icon="iconMode" class="iconfont mode" @click="changeMode"></font-awesome-icon>
                    </div>
                    <div class="icon i-left" :class="disableCls" >
                    <!-- <i class="iconfont icon-prev" @click="prev"></i> -->
                        <font-awesome-icon @click="prev" icon="step-backward" class="iconfont icon-prev"></font-awesome-icon>
                    </div>
                    <div class="icon i-center" :class="disableCls">
                        <font-awesome-icon :icon="playIcon" class="iconfont" @click="togglePlaying" ></font-awesome-icon>
                    <!-- <i class="iconfont" @click="togglePlaying" :class="playIcon"></i> -->
                    </div>
                    <div class="icon i-right" :class="disableCls" >
                        <font-awesome-icon @click="next" icon="step-forward" class="iconfont icon-test"></font-awesome-icon>
                    <!-- <i class="iconfont icon-test" @click="next"></i> -->
                    </div>
                    <div class="icon i-right">
                        <font-awesome-icon icon="heart" :class="getFavorite(currentSong)" @click="toggleFavorite(currentSong)" ></font-awesome-icon>
                    <!-- <i class="iconfont"  @click="toggleFavorite(currentSong)" :class="getFavoriteIcon(currentSong)"></i> -->
                    </div>
              </div>
          </div>
      </div>
     </transition>
     <transition name="mini">
      <div class="mini-player" v-show="!fullScreen" @click="open" >
          <div class="icon" >
              <img :class="cdClass" width="40" height="40" :src="currentSong.image" >
          </div>
          <div class="text">
              <h2 class="name" v-html="currentSong.name"></h2>
              <p class="desc" v-html="currentSong.singer"></p>
          </div>
          <div class="control">
              <progress-circle :redius="radius" :percent="percent">
                <font-awesome-icon  :icon="playIcon" class="fa" @click.stop="togglePlaying" ></font-awesome-icon>
              </progress-circle>
          </div>
          <div class="control" @click.stop="showPlayList">
              <font-awesome-icon icon="server" class="iconfont-playlist"></font-awesome-icon>
          </div>
      </div>
     </transition>
     <play-list @stopMusic="stopMusic" ref="playlist"></play-list>
     <audio ref="audio" @ended="end" @canplay="ready" @error="error" @timeupdate="updateTime" autoplay  muted></audio>
  </div>
</template>

<script>
import {mapGetters,mapMutations, mapActions} from 'vuex'
import {getSong,getLyric} from 'api/song'
import ProgressBar from 'base/progress-bar/progress-bar'
import ProgressCircle from 'base/progress-circle/progress-circle'
import Lyric from 'lyric-parser'
import {playMode} from 'assets/js/config'
import {shuffile} from 'assets/js/util'
import Scroll from 'base/scroll/scroll'
import PlayList from 'components/playlist/playlist'
export default {
    data() {
        return {
            url:'',
            songready:false,
            currentTime:0,
            duration:0,
            radius: 32,
            currentLyric: null,  
            currentLineNum:0,
            currentShow:'cd',
            noLyric: false,
            playingLyric:''
        }
    },
    created(){
        this.touch={}
    },
    computed:{
        cdClass(){
            return this.playing ? 'play' :'play pause'
        },
        playIcon(){
            return this.playing ? "stop-circle" :"play-circle"
        },
        disableCls(){
            return this.songready ? '' : 'disable'
        },
        percent(){
            return this.currentTime / this.duration
        },
        ...mapGetters([
            'fullScreen',
            'playlist',
            'currentSong',
            'playing',
            'currentIndex',
            'mode',
            'sequenceList',
            'favoriteList',
            'playHistory'
        ]),
        iconMode(){
            return this.mode ===playMode.sequence ? 'redo' : this.mode===playMode.loop ? 'sync' : 'random'
        },
        upDatecurrentLyric(){
            if (this.noLyric) {
                return '暂无歌词'
            }
            if (!this.noLyric) {
                return '歌词加载中'
            }
        }

    },
    methods:{
        back(){
            this.setFullScreen(false)
            this.currentShow='cd'
        },
        open(){
            this.setFullScreen(true)
        },
        end(){
            if(this.mode === playMode.loop){
                this.loop()
            }else{
                this.next()
            }
        },
        stopMusic(){
            this.$refs.audio.pause()
        },
        loop(){
            this.$refs.audio.currentTime=0
            this.$refs.audio.play()
            if(this.currentLyric){
                this.currentLyric.seek()
            }
        },
        showPlayList(){
            this.$refs.playlist.show()
        },
        toggleFavorite(song){
            if(this.isFavorite(song)){
                this.deleteFavoriteList(song)
            }
            else{
                this.saveFavoriteList(song)
            }
        },
        isFavorite(song){
            const index = this.favoriteList.findIndex((item)=>{
               return item.id === song.id 
            })
            return index > -1
        },
        getFavorite(song){
            if(this.isFavorite(song)){
                return 'iconfont icon-like'
            }else{
                return 'iconfont icon-dislike'
            }
        },
        ...mapMutations({
            setFullScreen:'SET_FULL_SCREEN',
            setPlayingState : 'SET_PLAYING_STATE',
            setCurrentIndex :'SET_CURRENT_INDEX',
            setPlayMode:'SET_PLAY_MODE',
            setPlayList:'SET_PLAYLIST'
        }),
        
        ...mapActions([
            'saveFavoriteList',
            'deleteFavoriteList',
            'saveHestoryList'
        ]),
        _getSong(id){
            getSong(id).then((res)=>{
                if(res.data.data[0].url!==null){
                    this.url=res.data.data[0].url
                }else{
                    this.next()
                    alert("歌曲不存在")
                }
            })   
        },
        _getLyric(id){ //注意这里如果用的是网易云的歌词，需要将lyric.js里的this.lines.push的time语句改为time: result[1] * 60 * 1000 +result[2] * 1000 +(parseInt(result[3]) || 0),
            if(this.currentLyric){
                this.currentLyric.stop()
                this.currentLyric=null
            }
            this.noLyric=false
            getLyric(id).then((res)=>{
                this.currentLyric = new Lyric(res.data.lrc.lyric,this.handleLyric)
                if(this.playing){
                    this.currentLyric.play()
                    this.currentLineNum = 0
                    this.$refs.lyricList.scrollTo(0, 0, 1000)
                }
            }).catch(()=>{
                this.currentLyric=null
                this.noLyric=true
                this.currentLineNum=0
            })
            
        },
        handleLyric ({lineNum, txt}) {
            this.currentLineNum = lineNum
            if (lineNum > 5) {//大于五行开始持续滚动，让显示歌词保持在中间
                let lineEl = this.$refs.lyricline[lineNum - 5]//当前播放的歌词
                this.$refs.lyricList.scrollToElement(lineEl, 1000)
            } else {
                this.$refs.lyricList.scrollTo(0, 0, 1000)
            }
            this.playingLyric= txt
        },
        togglePlaying(){
             if(!this.songready){
                return 
            }
            // const audio = this.$refs.audio
            this.setPlayingState(!this.playing)
            // this.playing ? audio.play() : audio.pause()
            if(this.currentLyric)
            {
                this.currentLyric.togglePlay()
            }
        },
        next(){
            if(!this.songready){
                return 
            }
            if(this.playlist.length===1){
                this.loop()
            }else{
            let index = this.currentIndex+1
            if(index === this.playlist.length){
                index =0
            }
            this.setCurrentIndex(index)
            if(!this.playing){
                this.togglePlaying()
            }
            this.songready=false
            }
        },
        prev(){
             if(!this.songready){
                return 
            }
            if(this.playlist.length===1){
                this.loop()
            }else{
            let index = this.currentIndex-1
            if(index === -1){
                index = this.playlist.length-1
            }
            this.setCurrentIndex(index)
            if(!this.playing){
                this.togglePlaying()
            }
            this.songready=false
            }
        },
        ready(){
            this.songready=true
            this.saveHestoryList(this.currentSong)
        },
        error(){
            this.songready=true
        },
        updateTime(e){
            this.currentTime=e.target.currentTime
        },
        format(interval){
            interval=interval | 0
            let minute = interval/60 |0
            let second = interval % 60
            if(second < 10){
               second = '0' + second
            }
            return minute+':'+second
        },
        onProgressBarChange(percent){
            const currentTime=this.duration * percent
            this.$refs.audio.currentTime= currentTime //整体时间乘以百分比
            if(!this.playing){
                this.togglePlaying()
            }
            if(this.currentLyric){
                const curent = currentTime * 1000
                this.currentLyric.seek(curent)
            }
        },
        changeMode(){
            const mode = (this.mode+1)%3
            this.setPlayMode(mode)
            let list = null
            if(mode ===playMode.random){
                list =shuffile(this.sequenceList)
            }else{
                list =this.sequenceList
            }

            this._resetCurrentIndex(list)
            this.setPlayList(list)
        },
        _resetCurrentIndex(list){
            let index = list.findIndex((item)=>{
                return item.id===this.currentSong.id
            })
            this.setCurrentIndex(index)
        },
        changeMiddle(){
             if (this.currentShow === 'cd') {
                this.currentShow = 'lyric'
            } else {
                this.currentShow = 'cd'
            }
        }
    },
    watch:{
        currentSong(newVal,oldVal){
            if(!newVal.id){
                return
            }
            if(newVal.id ===oldVal.id){
                return
            }
            this.$refs.audio.pause()
            this.$refs.audio.currentTime = 0
            this._getSong(newVal.id)
        },
        url(newUrl){
            this._getLyric(this.currentSong.id)
            this.playingLyric=''
            this.$refs.audio.src=newUrl
            let stop = setInterval(() => {
                this.duration = this.$refs.audio.duration //音频时长长度
                if (this.duration) {
                    clearInterval(stop)
                }
            }, 150)
            this.setPlayingState(true)
            
        },
        playing(newPlaying){
            const audio =this.$refs.audio
            newPlaying?audio.play() : audio.pause()
        }
    },
    components:{
        ProgressBar,
        ProgressCircle,
        Scroll,
        PlayList
    }
}
</script>

<style lang="stylus" scoped>
@import "~assets/stylus/variable"
@import "~assets/stylus/mixin"

.player
    .normal-player
        position :fixed
        left: 0
        right: 0
        top: 0
        bottom: 0
        z-index: 150
        background: $color-background
        .background
            position :absolute
            left:-50%
            top:-50%
            width:300%
            height:300%
            z-index:-1
            opacity :0.6
            filter:blur(30px)
            .filter
                position :absolute
                width:100%
                height:100%
                background:black
                opacity :0.6
        .top
            position:relative
            margin-bottom :25px
            .back
                position:absolute
                top:0
                left:6px
                z-index:50
                .icon
                    display:block
                    padding:5px 9px
                    font-size:35px
                    color:$color-theme-l
            .title
                width: 70%
                margin: 0 auto
                padding-top: 10px
                line-height: 20px
                text-align: center
                no-wrap()
                font-size: $font-size-large
                font-weight: bold
                color: $color-text-l
            .subtitle
                width: 70%
                margin: 0 auto
                line-height: 20px
                text-align: center
                no-wrap()
                font-size: $font-size-small-x
                color: $color-text-l
        .middle
            display: flex
            align-items: center
            position: fixed
            width: 100%
            top: 80px
            bottom: 170px
            white-space: nowrap
            font-size: 0
            .middle-l
                display: inline-block
                vertical-align: top
                position: relative
                width: 100%
                height: 0
                padding-top: 80%
                &.middleL-enter-active, &.middleL-leave-active 
                    transition: all 0.3s
                &.middleL-enter, &.middleL-leave-to 
                    opacity: 0
                .cd-wrapper
                    position: absolute
                    left: 10%
                    top: 0
                    width: 80%
                    height: 100%
                    .cd
                        width: 100%
                        height: 100%
                        box-sizing: border-box
                        border: 15px solid rgba(255, 255, 255, 0.1)
                        border-radius: 50%
                        &.play
                            animation: rotate 20s linear infinite
                        &.pause
                            animation-play-state: paused;
                        .image
                            position: absolute
                            left: 0
                            top: 0
                            width: 100%
                            height: 100%
                            border-radius: 50%
                .playing-lyric-wrapper
                    width: 80%
                    margin: 30px auto 0 auto
                    overflow: hidden
                    text-align: center
                    .playing-lyric
                        height: 20px
                        line-height: 20px
                        font-size: $font-size-medium
                        color: $color-text-l            
            .middle-r
                display: inline-block
                position: absolute
                top: 0
                vertical-align: top
                width: 100%
                height: 100%
                overflow: hidden
                &.middleR-enter-active, &.middleR-leave-active
                    transition: all 0.2s
                &.middleR-enter, &.middleR-leave-to 
                    opacity: 0
                .lyric-wrapper
                    width: 80%
                    margin: 0 auto
                    overflow: hidden
                    text-align: center
                    .text
                        line-height :40px
                        color:$color-text-ggg
                        font-size:$font-size-medium
                        &.current
                            color:#FFF
                    .no-lyric
                        line-height: 40px
                        margin-top: 60%
                        color: $color-text-ggg
                        font-size: $font-size-medium
        .bottom
            position:absolute
            bottom:50px
            width:100%
            .dot-wrapper
                text-align: center
                font-size: 0
                .dot
                    display: inline-block
                    vertical-align: middle
                    margin: 0 4px
                    width: 8px
                    height: 8px
                    border-radius: 50%
                    background: $color-text-l
                    &.active
                        width: 20px
                        border-radius: 5px
                        background: $color-text-ll
            .progress-wrapper
                display :flex
                align-items: center
                width: 80%
                margin: 0px auto
                padding: 10px 0
                .time
                    color:$color-text-l
                    font-size:$font-size-small
                    flex:0 0 30px
                    line-height:30px
                    widows 30px
                    &.time-l
                        text-align :left
                    &.time-r
                        text-align:right
                        color:$color-text-gg
                .progress-bar-wrapper
                    flex:1
            .operators
                display: flex
                align-items: center
                .icon
                    flex:1
                    color:$color-theme-l
                    &.disable
                        color:$color-theme-g
                    .iconfont
                        font-size:30px
                    .mode
                        font-size:25px
                    &.i-left
                        text-align :right
                    &.i-center
                        padding:0 20px
                        text-align :center
                        .iconfont
                            font-size:40px
                    &.i-right
                        text-align :left
                    .icon-like
                        font-size:30px
                        color:$color-sub-theme
                    .icon-dislick
                        font-size:30px
        &.normal-enter-active, &.normal-leave-active
            transition :all 0.4s
            .top,.bottom
                transition :all 0.4s cubic-bezier(0.86,0.18,0.82,1.32)
        &.normal-enter,&.normal-leave-to
            opacity :0
            .top
                transform :translate3d(0,-100px,0)
            .bottom
                transform :translate3d(0,100px,0)            
    .mini-player
        display:flex
        align-items :center
        position :fixed
        left :0
        bottom:0
        z-index: 180
        width: 100%
        height: 60px
        background: rgba(255, 255, 255, 0.85)
        &.mini-enter-active,&.mini-leave-active
            transition :all 0.4s
        &.mini-enter,&.mini-leave-to
            opacity :0
        .icon
            flex: 0 0 40px  
            width: 40px
            padding: 0 10px 0 20px
            img
                border-radius :50%
                &.play
                    animation: rotate 20s linear infinite
                &.pause
                    animation-play-state: paused;
        .text
            display: flex
            flex-direction: column
            justify-content: center
            flex: 1
            overflow: hidden
            .name
                margin-bottom: 2px
                line-height: 14px
                no-wrap()
                font-size: $font-size-medium
                color: $color-text
            .desc
                no-wrap();
                font-size: $font-size-small;
                color: $color-text;
        .control
            flex:0 0 30px
            width:30px
            padding : 0 10px
            .iconfont-playlist,.icon-font
                font-size: 30px;
                color: $color-theme-d;
                position:relative
                left:-5px
                top:-1.5px
            .fa
                color: $color-theme-d;
                font-size: 25px;
                position: absolute;
                left: 4px;
                top: 3.5px;
            
@keyframes rotate
    0%
        transform:rotate(0)
    100%
        transform :rotate(360deg)

</style>