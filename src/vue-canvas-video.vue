<template>
    <div>

        <canvas ref="canvas" :style="{width:width+'px',height:height+'px'}">
        </canvas>
        <div v-if="video&&video.poster&&!frame" style="position: absolute;top:0;left:0;background-size: cover"
             :style="{'background-image': 'url('+video.poster+')',width:width+'px',height:height+'px'}"></div>
    </div>
</template>


/*!
* vue-keyframe-animation v0.0.1 (https://github.com/johnnyGoo/vue-movieclip)
* Author: Johnny chen
*
* Copyright 2013-2016 Johnny chen
*/
<script>

    import WebGL2D from './webgl2d'

    let canvas = !!window.CanvasRenderingContext2D

    let webgl = (function () {
        try {

            var canvas = document.createElement('canvas');
            return !!(window.WebGLRenderingContext && (canvas.getContext('webgl') || canvas.getContext('experimental-webgl')));
        } catch (e) {
            return false;
        }

    })()


    function parseInt(str) {
        return Number(str.replace('px', ''));
    }

    // 注册
    export default {
        name: 'vue-canvas-video',
        // 声明 props
        props: {
            debug: {
                type: Boolean,
                default: false
            },


            type: {
                type: String,
                default: 'canvas'
            },
            render: {
                type: Boolean,
                default: true
            }
        },
        data: function () {
            return {
                playing: false, lastFrame: 0, frame: 0, iv: 0, totalFrame: 0, ctx: null, canvas: null, frames: [],
                width: 0, height: 0, __count: 0, audio_time: 0, inited: false, video: null,
                buffering: false
            }
        },
        methods: {

            loadClip: function (clip, buffering) {
                let self = this;


                if (!clip.img) {
                    let img = new Image();
                    img.src = clip.src;
                    img.crossOrigin = "Anonymous";

                    clip.img = img
                }
                if (buffering === true&&!self.loaded) {
                    self.buffering = buffering;
                    this.$emit('buffering', this);
                    clearTimeout(self.iv);

                }
                clip.img.onload = function () {
                    self.onLoadClip(clip)
                }
                if (this.debug) {
                    console.log('buffering ' + clip.src)
                }
            },
            onLoadClip: function (clip) {
                let self = this;
                if (this.debug) {
                    console.log('buffered  ' + clip.src)
                }
                clip.loaded = true;
                if (self.buffering === true) {
                    self.buffering = false;
                    this.$emit('buffered', this);

                    if (this.playing === true) {
                        self.nextFrame();
                    }

                }
                this.ctx.drawImage(clip.img, 0, 0, 0, 0);
            },

            loadNextClip: function (clip) {
                for (let i = 0; i < this.video.clips.length; i++) {
                    if (this.video.clips[i] === clip) {
                        if (i + 1 < this.video.clips.length) {
                            if (!this.video.clips[i + 1].img) {
                                this.loadClip(this.video.clips[i + 1], false)
                            }
                        }
                    }
                }
            }
            ,

            updateFrame: function (val) {
                if (this.inited === true && this.$el && this.buffering === false) {
                    if (val < 1 || val > this.totalFrame) {
                        return false
                    }

                    let clip = this.frames[val - 1].clip;

                    if (!clip.loaded) {
                        this.loadClip(clip, true)

                        return false;
                    }

                    this.loadNextClip(clip)



                    if (this.render) {
                        switch (this.type) {
                            case 'dom':
                                this.$el.children[0].children[this.lastFrame - 1].style.setProperty('display', 'none');
                                this.$el.children[0].children[val - 1].style.setProperty('display', 'block');
                                break;
                            case 'canvas':
                            case 'webgl':
                                let frame = this.frames[val - 1];

                                // this.ctx.clearRect(0, 0, this.width, this.height);
                                //  console.log(frame.x, frame.y, this.width, this.height*2, 0, 0, this.width, this.height)
                                this.ctx.drawImage(frame.clip.img, frame.x, frame.y, this.width, this.height, 0, 0, this.width, this.height);
                                break;

                        }
                        this.lastFrame = val;
                    }
                    this.$emit('frame', this);
                }
            },
            nextFrame: function () {
                if (this.inited === true) {
                    let nextframe;

                    if (this.video.forward === true) {
                        nextframe = this.goFixFrame(this.frame + 1)
                        if (nextframe === this.totalFrame) {
                            this.__count++
                            if (this.video.loop === this.__count) {
                                this.stop();
                            } else {
                                if (this.debug) {
                                    console.log('loop:' + this.__count)
                                }
                                if (this.video && this.video.audio) {
                                    this.video.audio.currentTime = 0
                                    this.video.audio.play()
                                }
                                this.autoNext();
                            }
                        } else {
                            this.autoNext();
                        }


                    } else {
                        nextframe = this.goFixFrame(this.frame - 1)
                        if (nextframe === 1) {
                            this.__count++
                            if (this.video.loop === this.__count) {
                                this.stop();
                            } else {
                                //循环
                                if (this.debug) {
                                    console.log('loop:' + this.__count)
                                }
                                if (this.video && this.video.audio) {
                                    this.video.audio.currentTime = 0
                                    this.video.audio.play()
                                }
                                this.autoNext();
                            }
                        } else {
                            this.autoNext();
                        }
                    }


                    if (this.video && this.video.audio !== null) {
                        if (Math.floor(this.video.audio.currentTime * 10) / 10 !== this.audio_time) {
                            this.audio_time = Math.floor(this.video.audio.currentTime * 10) / 10;
                            //console.log(this.audio_time+':'+Math.floor(this.audio_time*((1000/this.frameTime))))
                            this.frame = this.goFixFrame(Math.max(Math.floor(this.audio_time * ((1000 / this.frameTime))), 1))
                        } else {
                            this.frame = nextframe
                        }
                    } else {
                        this.frame = nextframe
                    }
                }


            },
            goFixFrame: function (frame) {

                while (frame < 1) {
                    frame = frame + this.totalFrame;
                }
                while (frame > this.totalFrame) {
                    frame = frame - this.totalFrame;
                }
                return frame
                //  this.frame = frame;
            },
            setFrameTime: function (time) {
                if (time >= 0) {
                    this.frameTime = time;
                }
            },
            skipTo: function (frame) {
                if (this.inited === true) {
                    this.frame = this.goFixFrame(frame);
                    if (this.video && this.video.audio) {
                        this.video.audio.currentTime = this.frame * this.frameTime / 1000
                    }

                }

            },
            autoNext: function () {
                clearTimeout(this.iv);

                this.iv = setTimeout(this.nextFrame, this.frameTime);
            },
            play: function () {
                if (this.inited === true) {
                    this.__count = 0;
                    this.playing = true;
                    this.frame = this.frame || 1;
                    this.nextFrame();
                    if (this.video && this.video.audio) {
                        this.video.audio.play();
                    }
                    // this.audio.play()
                    this.$emit('play', this);
                }

            },


            pause: function () {
                this.playing = false;
                if (this.video && this.video.audio) {
                    this.video.audio.pause()
                }
                clearTimeout(this.iv);
                this.$emit('pause', this);
            },
            resume: function () {
                if (this.playing === false && this.inited === true) {
                    this.playing = true;
                    if (this.video && this.video.audio) {
                        this.video.audio.play();
                    }
                    this.nextFrame();
                    this.$emit('resume', this);
                }
            },

            stop: function () {
                this.playing = false;
                clearTimeout(this.iv);
                if (this.video && this.video.audio) {
                    this.video.audio.pause();
                }
                this.$emit('stop', this);
            },

            initVideo: function (video) {
                if (!video) {
                    return;
                }
                this.playing = false;
                if (this.video && this.video.audio) {
                    this.video.audio.pause();
                }
                this.video = video;
                clearTimeout(this.iv);
                this.inited = false;
                this.frameTime = video.frameTime || 50;
                this.frame = 1
                this.lastFrame = 1

                if (this.video && this.video.audio) {
                    this.video.audio.currentTime = 0
                }

                this.width = video.width || this.width;
                this.height = video.height || this.height;


                this.__update_images();
            },
            __update_images: function () {
                let self = this;
                if (!this.video) {
                    return;
                }

                if (this.video.clips === null || this.video.clips.length === 0) {
                    return;
                }

                this.inited = true;
                this.frames = [];

                for (let i in this.video.clips) {
                    let clip = this.video.clips[i];

                    for (let f = 0; f < clip.clipFrames; f++) {
                        this.frames.push({
                            x: ((f) % clip.row) * self.width,
                            y: Math.floor((f) / clip.row) * self.height,
                            clip: clip
                        })
                    }

                }


                this.totalFrame = this.frames.length;
                this.initFrame = Math.max(0, Math.min(this.initFrame, this.totalFrame));
                this.frame = this.initFrame;
                this.__count = 0
                switch (this.type) {
                    case 'canvas':
                        this.canvas = this.$refs.canvas;
                        this.canvas.width = this.width;
                        this.canvas.height = this.height;
                        this.ctx = this.canvas.getContext("2d");
                        break
                    case 'webgl':
                        this.canvas = this.$refs.canvas;
                        this.canvas.width = this.width;
                        this.canvas.height = this.height;
                        WebGL2D.enable(this.canvas)
                        this.ctx = this.canvas.getContext("webgl-2d")


                }


                self.updateFrame(self.video.initFrame);


                //  this.updateFrame(this.frame);
                if (this.autoPlay) {
                    this.play();
                }
            }
        },

        watch: {
            width: function (val, oldVal) {
                try {
                    this.canvas.width = this.width;
                    this.updateFrame(this.frame)
                } catch (e) {
                    if (this.debug) {
                        console.log(e)
                    }
                }

            },
            height: function (val, oldVal) {
                try {
                    this.canvas.height = this.height;
                    this.updateFrame(this.frame)
                } catch (e) {
                    if (this.debug) {
                        console.log(e)
                    }
                }
            },
            frame: function (val, oldVal) {
                if (val !== oldVal) {
                    if (val > 0 && val <= this.totalFrame) {
                        this.updateFrame(val);
                    } else {
                        return false;
                    }
                }

            }
        },

        mounted: function () {

            this.initVideo(this.video)


        }


    }


</script>