<template>
    <div class="time-axis"
         :class="{disabled:disabled}">
        <canvas ref="cav"
                :style="canvasStyle"
                :width="width"
                :height="height">
        </canvas>
        <div ref="pointer"
             class="time-axis-pointer"
             :style="pointerStyle"
             @mousedown="mouseDown">
            <div class="handle"></div>
            <div class="line"></div>
            <div v-if="isDown"
                 class="tip"
                 :class="{left:leftAnchor,right:rightAnchor}">
                {{ timeText }}
            </div>
        </div>
    </div>
</template>
<script>
import Schema from './schema.js'
import Resize from './resize.js'
import { DateT } from '@digibird/common'
import '../../style/time-axis.less'

export default {
    props: {
        // 单个刻度的时长
        singleScaleDuration: {
            type: Number,
            default: 30 * 60 * 1000
        },
        // 可用轴列表
        timeAxisList: Array,
        value: {
            type: Number,
            default: 0
        },
        disabled: Boolean
    },
    data() {
        return {
            // 表盘显示的最大时长
            totalDuration: 24 * 60 * 60 * 1000,
            width: 0,
            height: 0,
            ctx: null,
            // 根据实际宽度计算出每个刻度的宽
            signalScaleWidth: 0,
            // 总刻度数量
            totalCell: 0,
            // 单个像素代表的时长
            signalPxDuration: 0,
            // 去除边距后的实际宽度
            realWidth: 0,
            // 轴距离顶点的距离
            axisTop: 16,
            // 文本距离轴的间距
            foutMarginTop: 8,
            style: {
                // 左边距
                paddingLeft: 0,
                // 右边距
                paddingRight: 0,
                // 长刻度线高
                longLine: 10,
                // 短刻度线高
                shortLine: 6,
                // 轴距离顶点的距离
                axisTop: 16,
                // 文本距离轴的间距
                foutMarginTop: 6,
                // 时间凹槽的高度
                timeSlotHeight: 6,
                // 时间凹槽的背景色
                timeSlotColor: '',
                // 可用轴的高度
                timeAxisHeight: 4,
                // 可用轴的背景色
                timeAxisColor: '',
                // 刻度颜色
                lineColor: '',
                // 字体颜色
                fontColor: '',
                // 字体样式
                font: ''
            },
            // 指针的宽
            pointerWidth: 0,
            offsetLeft: 0,
            isDown: false
        }
    },
    computed: {
        canvasStyle() {
            return {
                width: `${this.width}px`,
                height: `${this.height}px`
            }
        },
        pointerStyle() {
            return {
                left: `${this.pointerPosition}px`
            }
        },
        // 凌晨零点
        firstTime() {
            if (this.value === 0) {
                return new Date(new Date().toLocaleDateString()).getTime()
            }
            return new Date(new Date(this.value).toLocaleDateString()).getTime()
        },
        // 23:59:59
        lastTime() {
            return this.firstTime + this.totalDuration - 1
        },
        // 当前指针位置
        pointerPosition: {
            get() {
                let time = this.firstTime
                if (this.value > 0) {
                    time = this.value
                }
                return (time - this.firstTime) / this.signalPxDuration - this.pointerWidth / 2 + this.style.paddingLeft
            },
            set(val) {
                if (val === 0) {
                    this.$emit('input', this.firstTime)
                    return
                }
                if (val === this.realWidth) {
                    this.$emit('input', this.lastTime)
                    return
                }
                const time = val * this.signalPxDuration
                this.$emit('input', Math.round(this.firstTime + time))
            }
        },
        timeText() {
            return DateT.format(this.value, 'HH:mm')
        },
        leftAnchor() {
            return this.value - this.firstTime < 400000
        },
        rightAnchor() {
            return this.lastTime - this.value < 400000
        }
    },
    watch: {
        timeAxisList: 'resize'
    },
    mounted() {
        Resize.remove(this.resize)
        Resize.add(this.resize)
        this.destroyed()
        this.$nextTick(() => {
            this.setStyle()
            this.resize()
        })
    },
    methods: {
        resize() {
            const { width, height, left } = this.$el.getBoundingClientRect()
            this.offsetLeft = left
            this.pointerWidth = this.$refs.pointer.getBoundingClientRect().width
            this.width = width
            this.height = height
            this.realWidth = this.width - this.style.paddingLeft - this.style.paddingRight
            this.ctx = this.$refs.cav.getContext('2d')
            this.totalCell = this.totalDuration / this.singleScaleDuration
            this.signalScaleWidth = this.realWidth / this.totalCell
            this.signalPxDuration = this.totalDuration / this.realWidth
            setTimeout(() => {
                this.repaint()
            }, 300)
        },
        // 获取clss中的样式描述更改现有样式
        setStyle() {
            const style = document.defaultView.getComputedStyle(this.$el)
            Object.keys(Schema).forEach(key => {
                const val = style.getPropertyValue(`--${key}`).trim()
                if (!val) {
                    return false
                }
                if (Schema[key] === 'number') {
                    this.style[key] = parseInt(val, 10)
                    // this.$set(this.style, key, parseInt(val, 10))
                    return false
                }
                this.style[key] = val
                // this.$set(this.style, key, val)
                return false
            })
        },
        // 绘制线
        drawLine(beginX, beginY, endX, endY, width, color) {
            this.ctx.save()
            this.ctx.beginPath()
            this.ctx.moveTo(beginX + 0.5, beginY)
            this.ctx.lineTo(endX + 0.5, endY)
            this.ctx.strokeStyle = color
            this.ctx.lineWidth = width
            this.ctx.stroke()
            this.ctx.restore()
        },
        // 绘制文本
        drawText(text, x, y, maxWidth, color) {
            this.ctx.save()
            this.ctx.font = this.style.font
            this.ctx.fillStyle = color
            this.ctx.textAlign = 'center'
            this.ctx.textBaseline = 'top'
            this.ctx.fillText(text, x, y, maxWidth)
            this.ctx.restore()
        },
        // 绘制刻度尺
        repaint() {
            this.ctx.clearRect(0, 0, this.width, this.height)
            this.drawTimeSlot(this.style.paddingLeft, this.style.axisTop, this.realWidth, this.style.timeSlotHeight, this.style.timeSlotColor)
            const timeAxisTop = this.style.axisTop + (this.style.timeSlotHeight - this.style.timeAxisHeight) / 2
            this.drawTimeAxisList(timeAxisTop, this.style.timeAxisHeight, this.style.timeAxisColor)
            const lineY1 = this.style.axisTop + this.style.timeSlotHeight
            for (let i = 0; i < this.totalCell; i++) {
                const x = i * this.signalScaleWidth + this.style.paddingLeft
                let y = this.style.shortLine + lineY1
                const date = new Date(this.firstTime + this.signalPxDuration * i * this.signalScaleWidth)
                const minutes = date.getMinutes()
                if (minutes === 0) {
                    y = this.style.longLine + lineY1
                    this.drawText(DateT.format(date, 'HH:mm'), x + 0.5, y + this.style.foutMarginTop, this.signalScaleWidth, this.style.fontColor)
                }
                this.drawLine(x, lineY1, x, y, 1.5, this.style.lineColor)
            }
        },
        // 绘制已经存在的轴
        drawTimeAxisList(y, height, color) {
            if (!this.timeAxisList || this.timeAxisList.length === 0) return
            this.timeAxisList.forEach(item => {
                const { start, end, firstTime } = item
                const startTime = start - firstTime
                const endTime = end - firstTime
                const startPx = Math.round(startTime / this.signalPxDuration + this.style.paddingLeft)
                const endPx = Math.round(endTime / this.signalPxDuration + this.style.paddingLeft - startPx)
                this.drawTimeAxis(startPx, y, endPx, height, color)
            })
        },
        // 绘制时间凹槽
        drawTimeSlot(x, y, width, height, color) {
            this.ctx.fillStyle = color
            this.ctx.fillRect(x, y, width, height)
        },
        // 绘制填充的时间轴
        drawTimeAxis(x, y, w, h, color) {
            const hieght = h - 2
            this.ctx.fillStyle = color
            this.ctx.fillRect(x, y + 1, w, hieght)
        },
        mouseDown() {
            this.isDown = true
            this.$emit('down')
            this.destroyed()
            document.addEventListener('mousemove', this.move)
            document.addEventListener('mouseup', this.up)
        },
        move(e) {
            let newPos = e.clientX - this.offsetLeft - this.style.paddingLeft
            if (newPos <= 0) {
                newPos = 0
            }
            if (newPos > this.realWidth) {
                newPos = this.realWidth
            }
            this.pointerPosition = newPos
        },
        up(e) {
            this.isDown = false
            let newPos = e.clientX - this.offsetLeft - this.style.paddingLeft
            if (newPos <= 0) {
                newPos = 0
            }
            if (newPos > this.realWidth) {
                newPos = this.realWidth
            }
            if (newPos !== this.pointerPosition) {
                this.$emit('change', Math.round(this.firstTime + newPos * this.signalPxDuration))
            }
            this.pointerPosition = newPos
            this.destroyed()
        },
        destroyed() {
            document.removeEventListener('mousemove', this.move)
            document.removeEventListener('mouseup', this.up)
        }
    }
}
</script>
