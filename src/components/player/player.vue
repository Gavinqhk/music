<template>
    <div class="player" v-show="playlist.length>0">
      <transition name="normal"
                  @enter="enter"
                  @after-enter='afterEnter'
                  @leave="leave"
                  @after-leave="afterLeave">
        <div class="normal-player" v-show="fullScreen">
            <div class="background">
                <img width="100%" height="100%" :src="currentSong.image">
            </div>
            <div class="top">
                <div class="back" @click="back">
                    <i class="icon-back"></i>
                </div>
                <h1 class="title" v-html="currentSong.name"></h1>
                <h2 class="subtitle" v-html="currentSong.singer"></h2>
            </div>
            <div class="middle"
				@touchstart.prevent="onMiddleTouchStart"
				@touchmove.prevent="onMiddleTouchMove"
				@touchend="onMiddleTouchEnd">
                <div class="middle-l" ref="middleL">
                    <div class="cd-wrapper" ref="cdWrapper">
                        <div class="cd" ref="imageWrapper">
                            <img class="image" :class="cdCls" :src="currentSong.image">
                        </div>
                    </div>
                    <div class="playing-lyric-wrapper">
                      <div class="playing-lyric">{{playingLyric}}</div>
                    </div>
                </div>
                <scroll class="middle-r" ref="lyricList" :data="currentLyric && currentLyric.lines">
					<div class="lyric-wrapper">
						<div v-if="currentLyric">
						<p ref="lyricLine" class="text"
							:class="{'current' :currentLineNum === index}"
							v-for="(line, index ) in currentLyric.lines"
							:key="index">{{line.txt}}</p>
						</div>
					</div>
                </scroll>
            </div>
            <div class="bottom">
				<div class="dot-wrapper">
					<span class="dot" :class="{'active' : currentShow === 'cd'}"></span>
					<span class="dot" :class="{'active' : currentShow === 'lyric'}"></span>
				</div>
				<div class="progress-wrapper">
					<span class="time time-l">{{format(currentTime)}}</span>
					<div class="progress-bar-wrapper">
						<progress-bar ref="progressBar" @percentChange="onProgressBarChange" :percent="percent"></progress-bar>
					</div>
					<span class="time time-r">{{format(currentSong.duration)}}</span>
				</div>
                <div class="operators">
                    <div class="icon i-left">
                        <i :class="iconMode" @click="changeMode"></i>
                    </div>
                    <div class="icon i-left" :class="disableCls">
                        <i class="icon-prev" @click="preve"></i>
                    </div>
                    <div class="icon i-center">
                        <i :class="playIcon" @click="togglePlaying"></i>
                    </div>
                    <div class="icon i-right" :class="disableCls">
                        <i class="icon-next" @click="next"></i>
                    </div>
                    <div class="icon i-right">
						<i @click="toggleFavorite(currentSong)" class="icon" :class="favoriteIcon"></i>
                    </div>
                </div>
            </div>
        </div>
      </transition>
      <transition name="mini">
        <div class="mini-player" v-show="!fullScreen" @click="open">
            <div class="icon">
                <div class="imgWrapper" ref="miniWrapper">
                    <img ref="miniImage" width="40" height="40" :class="cdCls" :src="currentSong.image">
                </div>
            </div>
            <div class="text">
                <h2 class="name" v-html="currentSong.name"></h2>
                <p class="desc" v-html="currentSong.singer"></p>
            </div>
            <div class="control" @click.stop="togglePlaying">
				<progress-circle :radius="radius" :percent="percent">
					<i class="icon-mini" :class="miniplayIcon"></i>
				</progress-circle>
            </div>
            <div class="control" @click.stop="showPlaylist">
                <i class="icon-playlist"></i>
            </div>
        </div>
      </transition>
		<playlist ref="playlist"></playlist>
		<audio :src="currentSong.url" ref="audio" @playing="ready" @error="error" @timeupdate="updateTime" @ended="end"></audio>
    </div>
</template>
<script>
import { mapGetters, mapMutations, mapActions } from 'vuex'
import animations from 'create-keyframe-animation'
import { prefixStyle } from 'common/js/dom'
import { playMode } from 'common/js/config'
import { shuffle } from 'common/js/util'
import ProgressBar from 'base/progress-bar/progress-bar'
import ProgressCircle from 'base/progress-circle/progress-circle'
import Scroll from 'base/scroll/scroll'
import Playlist from 'components/playlist/playlist'
import Lyric from 'lyric-parser'
import { playerMixin } from 'common/js/mixin'

const transform = prefixStyle('transform')
const transitionDuration = prefixStyle('transitionDuration')

const timeExp = /\[(\d{2}):(\d{2}):(\d{2})]/g

export default {
	mixins: [playerMixin],
	data () {
		return {
			songReady: false,
			currentTime: 0,
			radius: 32,
			playingLyric: '',
			currentLyric: null,
			currentLineNum: 0,
			currentShow: 'cd',
			isPureMusic: false,
			pureMusicLyric: ''
		}
	},
	created () {
		this.touch = {}
	},
	computed: {
		...mapGetters([
			'fullScreen',
			'playlist',
			'currentSong',
			'playing',
			'currentIndex',
			'mode',
			'sequenceList'
		]),
		playIcon () {
			return this.playing ? 'icon-pause' : 'icon-play'
		},
		miniplayIcon () {
			return this.playing ? 'icon-pause-mini' : 'icon-play-mini'
		},
		cdCls () {
			return this.playing ? 'play' : ''
		},
		disableCls () {
			return this.songReady ? '' : 'disable'
		},
		percent () {
			return this.currentTime / this.currentSong.duration
		},
		iconMode () {
			return this.mode === playMode.sequence ? 'icon-sequence' : this.mode === playMode.loop ? 'icon-loop' : 'icon-random'
		}
	},
	methods: {
		...mapMutations({
			setFullScreen: 'SET_FULL_SCREEN',
			setPlaying: 'SET_PLAYING_STATE',
			setCurrentIndex: 'SET_CURRENT_INDEX',
			setPlayMode: 'SET_PLAY_MODE',
			setPlaylist: 'SET_PLAYLIST',
			setPlayingState: 'SET_PLAYING_STATE'
		}),
		...mapActions([
			'savePlayHistory'
		]),
		updateTime (e) {
			this.currentTime = e.target.currentTime
		},
		back () {
			this.setFullScreen(false)
		},
		open () {
			this.setFullScreen(true)
		},
		togglePlaying () {
			if (!this.songReady) {
				return
			}
			this.playing ? this.$refs.audio.pause() : this.$refs.audio.play()
			this.setPlaying(!this.playing)
			if (this.currentLyric) {
				this.currentLyric.togglePlay()
			}
		},
		changeMode () {
			const mode = (this.mode + 1) % 3
			this.setPlayMode(mode)
			let list = null
			if (mode === playMode.random) {
				list = shuffle(this.sequenceList)
			} else {
				list = this.sequenceList
			}
			this.resetCurrentIndex(list)
			this.setPlaylist(list)
		},
		resetCurrentIndex (list) {
			let index = list.findIndex((item) => {
				return item.id === this.currentSong.id
			})
			this.setCurrentIndex(index)
		},
		onProgressBarChange (percent) {
			const currentTime = this.currentSong.duration * percent
			this.currentTime = this.$refs.audio.currentTime = currentTime
			if (this.currentLyric) {
				this.currentLyric.seek(currentTime * 1000)
			}
			if (!this.playing) {
				this.togglePlaying()
			}
		},
		next () {
			if (!this.songReady) {
				return
			}
			let index = this.currentIndex + 1
			if (index === this.playlist.length) {
				index = 0
			}
			this.setCurrentIndex(index)
			if (!this.playing) {
				this.togglePlaying()
			}
			this.songReady = false
		},
		preve () {
			if (!this.songReady) {
				return
			}
			let index = this.currentIndex - 1
			if (index === -1) {
				index = this.playlist.length - 1
			}
			this.setCurrentIndex(index)
			if (!this.playing) {
				this.togglePlaying()
			}
			this.songReady = false
		},
		end () {
			this.currentTime = 0
			if (this.mode === playMode.loop) {
				this.loop()
			} else {
				this.next()
			}
		},
		loop () {
			this.$refs.audio.currentTime = 0
			this.$refs.audio.play()
			this.setPlayingState(true)
			if (this.currentLyric) {
				this.currentLyric.seek(0)
			}
		},
		ready () {
			this.songReady = true
			this.canLyricPlay = true
			this.savePlayHistory(this.currentSong)
		},
		error () {
			this.songReady = true
		},
		enter (el, done) {
			const { x, y, scale } = this._getPosAndScale()
			let animation = {
				0: {
				transform: `translate3d(${x}px, ${y}px, 0) scale(${scale})`
				},
				60: {
				transform: `translate3d(0, 0, 0) scale(1.1)`
				},
				100: {
				transform: `translate3d(0, 0, 0) scale(1)`
				}
			}
			animations.registerAnimation({
				name: 'move',
				animation,
				presets: {
				duration: 400,
				easing: 'linear'
				}
			})
			animations.runAnimation(this.$refs.cdWrapper, 'move', done)
		},
		afterEnter () {
			animations.unregisterAnimation('move')
			this.$refs.cdWrapper.style.animation = ''
		},
		leave (el, done) {
			this.$refs.cdWrapper.style.transition = 'all 0.4s'
			const { x, y, scale } = this._getPosAndScale()
			this.$refs.cdWrapper.style[transform] = `translate3d(${x}px, ${y}px, 0) scale(${scale})`
			this.$refs.cdWrapper.addEventListener('transitionend', done)
		},
		afterLeave () {
			this.$refs.cdWrapper.style.transition = ''
			this.$refs.cdWrapper.style[transform] = ''
		},
		format (interval) {
			interval = interval | 0
			const minute = interval / 60 | 0
			const second = this._pad(interval % 60, 2)
			return `${minute}:${second}`
		},
		getLyric () {
			this.currentSong.getLyric().then((lyric) => {
				if (this.currentSong.lyric !== lyric) {
					return
				}
				this.currentLyric = new Lyric(lyric, this.handleLyric)
				this.isPureMusic = !this.currentLyric.lines.length
				if (this.isPureMusic) {
					this.pureMusicLyric = this.currentLyric.lrc.replace(timeExp, '').trim()
					this.playingLyric = this.pureMusicLyric
				} else {
					if (this.playing) {
						// 这个时候有可能用户已经播放了歌曲，要切到对应位置
						this.currentLyric.seek(this.currentTime * 1000)
					}
				}
				}).catch(() => {
					this.currentLyric = null
					this.playingLyric = ''
					this.currentLineNum = 0
			})
		},
		handleLyric ({ lineNum, txt }) {
			if (!this.$refs.lyricLine) {
				return
			}
				this.currentLineNum = lineNum
			if (lineNum > 5) {
				let lineEl = this.$refs.lyricLine[lineNum - 5]
				this.$refs.lyricList.scrollToElement(lineEl, 1000)
			} else {
				this.$refs.lyricList.scrollTo(0, 0, 1000)
			}
			this.playingLyric = txt
		},
        showPlaylist () {
            this.$refs.playlist.show()
        },
		onMiddleTouchStart (e) {
			this.touch.initiated = true
			// 用来判断是否是一次移动
			this.touch.moved = false
			this.touch.startX = e.touches[0].pageX
			this.touch.startY = e.touches[0].pageY
		},
		onMiddleTouchMove (e) {
			if (!this.touch.initiated) {
				return
			}
			const touch = e.touches[0]
			const deltaX = touch.pageX - this.touch.startX
			const deltaY = touch.pageY - this.touch.startY
			if (Math.abs(deltaY) > Math.abs(deltaX)) {
				return
			}
			const left = this.currentShow === 'cd' ? 0 : -window.innerWidth
			const offsetWidth = Math.min(0, Math.max(-window.innerWidth, left + deltaX))
			this.touch.percent = Math.abs(offsetWidth / window.innerWidth)
			this.$refs.lyricList.$el.style[transform] = `translate3d(${offsetWidth}px, 0, 0)`
			this.$refs.lyricList.$el.style[transitionDuration] = 0
			this.$refs.middleL.style.opacity = 1 - this.touch.percent
		},
		onMiddleTouchEnd () {
			let offsetWidth
			let opacity
			if (this.currentShow === 'cd') {
				if (this.touch.percent > 0.1) {
					offsetWidth = -window.innerWidth
					this.touch.percent = 1
					this.currentShow = 'lyric'
					opacity = 0
				} else {
					this.touch.percent = 0
					offsetWidth = 0
					opacity = 1
				}
			} else {
				if (this.touch.percent < 0.9) {
					offsetWidth = 0
					this.touch.percent = 0
					opacity = 1
					this.currentShow = 'cd'
				} else {
					offsetWidth = -window.innerWidth
					this.touch.percent = 1
					opacity = 0
				}
			}
			this.$refs.lyricList.$el.style[transform] = `translate3d(${offsetWidth}px, 0, 0)`
			this.$refs.lyricList.$el.style[transitionDuration] = '300ms'
			this.$refs.middleL.style.opacity = opacity
			this.$refs.middleL.style[transitionDuration] = '300ms'
		},
		_pad (num, n) {
			let length = num.toString().length
			while (length < n) {
				num = '0' + num
				length++
			}
			return num
		},
		_getPosAndScale () {
			const targetWidth = 40
			const paddingTop = 80
			const paddingLeft = 40
			const paddingBottom = 30
			const width = window.innerWidth * 0.8
			const scale = targetWidth / width
			// x,y 是移动距离
			const x = -(window.innerWidth / 2 - paddingLeft)
			const y = window.innerHeight - paddingTop - width / 2 - paddingBottom
			return {
				x: x,
				y: y,
				scale: scale
			}
		}
	},
	watch: {
		currentSong (newSong, oldSong) {
			if (oldSong.id === newSong.id) {
				return
			}
			this.songReady = false
			this.canLyricPlay = false
			if (this.currentLyric) {
				this.currentLyric.stop()
				// 重置为null
				this.currentLyric = null
				this.currentTime = 0
				this.playingLyric = ''
				this.currentLineNum = 0
			}
			this.$nextTick(() => {
				this.$refs.audio.play()
				this.getLyric()
			}, 1000)
		}
	},
	components: {
		ProgressBar,
		ProgressCircle,
		Scroll,
		Playlist
	}
}
</script>
<style scoped lang="stylus" rel="stylesheet/stylus">
  @import "~common/stylus/variable"
  @import "~common/stylus/mixin"

  .player
    .normal-player
      position: fixed
      left: 0
      right: 0
      top: 0
      bottom: 0
      z-index: 150
      background: $color-background
      .background
        position: absolute
        left: 0
        top: 0
        width: 100%
        height: 100%
        z-index: -1
        opacity: 0.6
        filter: blur(20px)
      .top
        position: relative
        margin-bottom: 25px
        .back
          position absolute
          top: 0
          left: 6px
          z-index: 50
          .icon-back
            display: block
            padding: 9px
            font-size: $font-size-large-x
            color: $color-theme
            transform: rotate(-90deg)
        .title
          width: 70%
          margin: 0 auto
          line-height: 40px
          text-align: center
          no-wrap()
          font-size: $font-size-large
          color: $color-text
        .subtitle
          line-height: 20px
          text-align: center
          no-wrap()
          font-size: $font-size-medium
          color: $color-text
      .middle
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
          .cd-wrapper
            position: absolute
            left: 10%
            top: 0
            width: 80%
            box-sizing: border-box
            height: 100%
            .cd
              width: 100%
              height: 100%
              border-radius: 50%
              .image
                position: absolute
                left: 0
                top: 0
                width: 100%
                height: 100%
                box-sizing: border-box
                border-radius: 50%
                border: 10px solid rgba(255, 255, 255, 0.1)
              .play
                animation: rotate 20s linear infinite
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
          vertical-align: top
          width: 100%
          height: 100%
          overflow: hidden
          .lyric-wrapper
            width: 80%
            margin: 0 auto
            overflow: hidden
            text-align: center
            .text
              line-height: 32px
              color: $color-text-l
              font-size: $font-size-medium
              &.current
                color: $color-text
            .pure-music
              padding-top: 50%
              line-height: 32px
              color: $color-text-l
              font-size: $font-size-medium
      .bottom
        position: absolute
        bottom: 50px
        width: 100%
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
          display: flex
          align-items: center
          width: 80%
          margin: 0px auto
          padding: 10px 0
          .time
            color: $color-text
            font-size: $font-size-small
            flex: 0 0 30px
            line-height: 30px
            width: 30px
            &.time-l
              text-align: left
            &.time-r
              text-align: right
          .progress-bar-wrapper
            flex: 1
        .operators
          display: flex
          align-items: center
          .icon
            flex: 1
            color: $color-theme
            &.disable
              color: $color-theme-d
            i
              font-size: 30px
          .i-left
            text-align: right
          .i-center
            padding: 0 20px
            text-align: center
            i
              font-size: 40px
          .i-right
            text-align: left
          .icon-favorite
            color: $color-sub-theme
      &.normal-enter-active, &.normal-leave-active
        transition: all 0.4s
        .top, .bottom
          transition: all 0.4s cubic-bezier(0.86, 0.18, 0.82, 1.32)
      &.normal-enter, &.normal-leave-to
        opacity: 0
        .top
          transform: translate3d(0, -100px, 0)
        .bottom
          transform: translate3d(0, 100px, 0)
    .mini-player
      display: flex
      align-items: center
      position: fixed
      left: 0
      bottom: 0
      z-index: 180
      width: 100%
      height: 60px
      background: $color-highlight-background
      &.mini-enter-active, &.mini-leave-active
        transition: all 0.4s
      &.mini-enter, &.mini-leave-to
        opacity: 0
      .icon
        flex: 0 0 40px
        width: 40px
        height: 40px
        padding: 0 10px 0 20px
        .imgWrapper
          height: 100%
          width: 100%
          img
            border-radius: 50%
            &.play
              animation: rotate 10s linear infinite
            &.pause
              animation-play-state: paused
      .text
        display: flex
        flex-direction: column
        justify-content: center
        flex: 1
        line-height: 20px
        overflow: hidden
        .name
          margin-bottom: 2px
          no-wrap()
          font-size: $font-size-medium
          color: $color-text
        .desc
          no-wrap()
          font-size: $font-size-small
          color: $color-text-d
      .control
        flex: 0 0 30px
        width: 30px
        padding: 0 10px
        .icon-play-mini, .icon-pause-mini, .icon-playlist
          font-size: 30px
          color: $color-theme-d
        .icon-mini
          font-size: 32px
          position: absolute
          left: 0
          top: 0

  @keyframes rotate
    0%
      transform: rotate(0)
    100%
      transform: rotate(360deg)
</style>