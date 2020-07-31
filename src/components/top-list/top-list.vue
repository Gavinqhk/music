<template>
    <transition appear name="slider">
        <music-list :rank="rank" :songs="songs" :title="title" :bg-image="bgImage"></music-list>
    </transition>
</template>
<script>
import { getMusicList } from 'api/rank'
import { mapGetters } from 'vuex'
import { ERR_OK } from 'api/config'
import { createSong, isValidMusic, processSongsUrl } from 'common/js/song'
import MusicList from 'components/music-list/music-list'

export default {
    computed: {
        title () {
            return this.toplist.topTitle
        },
        bgImage () {
            return this.toplist.picUrl
        },
        ...mapGetters([
            'toplist'
        ])
    },
    data () {
        return {
            songs: [],
            rank: true
        }
    },
    created () {
        this._getMusicList()
    },
    methods: {
        _getMusicList () {
            if (!this.toplist.id) {
                this.$router.push('/rank')
                return
            }
            getMusicList(this.toplist.id).then((res) => {
                if (res.code === ERR_OK) {
                    processSongsUrl(this._normalizeSongs(res.songlist)).then((songs) => {
                        this.songs = songs
                    })
                }
            })
        },
        _normalizeSongs (list) {
            let ret = []
            list.forEach((item) => {
                const musicData = item.data
                if (isValidMusic(musicData)) {
                    ret.push(createSong(musicData))
                }
            })
            return ret
        }
    },
    components: {
        MusicList
    }
}
</script>
<style lang="stylus" scoped>
    .slider-enter-active, .slider-leave-active
        transition: all 0.3s

    .slider-enter, .slider-leave-to
        transform: translate3d(100%, 0, 0)
</style>
