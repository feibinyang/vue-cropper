<template>
    <div :class="`${$options.name}-container`"
        :style="{width: `${width}px`, height: `${height}px`}">
        <div :class="`${$options.name}-wrap-box`"><img :src="url"></div>
        <div :class="`${$options.name}-modal-box`"></div>
        <div :class="`${$options.name}-crop-box`"
            :style="{width: `${cropWidth}px`, height: `${cropHeight}px`,
            transform:`translate(${translateX}px, ${translateY}px)`}"
            @mousedown="cropStart">
            <span :class="`${$options.name}-view-box`">
                <img :src="url" :style="{width: `${width}px`, height: `${height}px`,
                    transform:`translate(-${translateX}px, -${translateY}px)`}">
            </span>
            <span :class="`${$options.name}-dashed dashed-h`"></span>
            <span :class="`${$options.name}-dashed dashed-v`"></span>
            <span :class="`${$options.name}-center`"></span>
            <span :class="`${$options.name}-face`" data-action="move"></span>
            <span :class="`${$options.name}-line line-e`" data-action="e"></span>
            <span :class="`${$options.name}-line line-n`" data-action="n"></span>
            <span :class="`${$options.name}-line line-w`" data-action="w"></span>
            <span :class="`${$options.name}-line line-s`" data-action="s"></span>
            <span :class="`${$options.name}-point point-e`" data-action="e"></span>
            <span :class="`${$options.name}-point point-n`" data-action="n"></span>
            <span :class="`${$options.name}-point point-w`" data-action="w"></span>
            <span :class="`${$options.name}-point point-s`" data-action="s"></span>
            <span :class="`${$options.name}-point point-ne`" data-action="ne"></span>
            <span :class="`${$options.name}-point point-nw`" data-action="nw"></span>
            <span :class="`${$options.name}-point point-sw`" data-action="sw"></span>
            <span :class="`${$options.name}-point point-se`" data-action="se"></span>
        </div>
    </div>
</template>
<script>
import {bind, assign, pick} from 'lodash';

export default {
    name: 'cropper',
    model: {
        prop: 'clipData',
        event: 'update'
    },
    props: {
        url: String,
        ratio: { // 裁剪比例
            type: Number,
            default: 1 / 1
        },
        area: { // 裁剪区域大小
            type: Number,
            default: .7,
            validator(value) {
                return value > 0 && value <= 1;
            }
        },
        width: Number,
        height: Number,
        clipData: {
            type: Object,
            default() {
                return {};
            }
        }
    },
    data() {
        return {
            cropWidth: 0,
            cropHeight: 0,
            translateX: 0,
            translateY: 0,
            naturalWidth: 0,
            naturalHeight: 0
        };
    },
    computed: {
        localRatio() {
            return this.ratio;
        }
    },
    watch: {
        ratio(value) {
            this.localRatio = value;
            this.setSize();
        }
    },
    async created() {
        this.setSize();
        document.addEventListener('mousemove',
            (this.cropping = bind(this.cropping, this)), false);
        document.addEventListener('mouseup',
            (this.cropEnd = bind(this.cropEnd, this)), false);
        await this.getNatureSize();
    },
    destroyed() {
        document.removeEventListener('mousemove', this.cropping, false);
        document.removeEventListener('mouseup', this.cropEnd, false);
    },
    methods: {
        setSize() {
            this.cropWidth = this.width * this.area;
            // 不限制宽高比
            if (this.localRatio === 0) {
                this.cropHeight = this.height * this.area;
            }
            else {
                this.cropHeight = Math.min(this.height, this.cropWidth / this.localRatio);
                this.cropWidth = this.cropHeight * this.localRatio;
            }
            this.translateX = (this.width - this.cropWidth) / 2;
            this.translateY = (this.height - this.cropHeight) / 2;
        },
        getNatureSize() {
            return new Promise((resolve) => {
                let image = document.createElement('img');
                image.onload = () => {
                    image.onload = image.onerror = null;
                    this.naturalWidth = image.naturalWidth || this.width;
                    this.naturalHeight = image.naturalHeight || this.height;
                    resolve();
                };
                image.onerror = () => {
                    image.onload = image.onerror = null;
                    this.naturalWidth = this.width;
                    this.naturalHeight = this.height;
                    resolve();
                };
                image.src = this.url;
            });
        },
        cropStart(e) {
            let cropElement = e.target || e.srcElement;
            assign(this, pick(e, ['pageX', 'pageY']));
            this.action = cropElement.getAttribute('data-action');
        },
        cropping(e) {
            if (!this.action) {
                return;
            }
            let gapX = e.pageX - this.pageX;
            let gapY = e.pageY - this.pageY;
            assign(this, pick(e, ['pageX', 'pageY']));
            if (this.action === 'move') {
                this.move(gapX, gapY);
            }
            else {
                this.resize(gapX, gapY);
            }
        },
        move(gapX, gapY) {
            this.translateX = Math.max(0, Math.min(
                this.width - this.cropWidth,
                this.translateX + gapX)
            );
            this.translateY = Math.max(0, Math.min(
                this.height - this.cropHeight,
                this.translateY + gapY)
            );
        },
        resize(gapX, gapY) {
            let {
                translateX: left,
                translateY: top,
                cropWidth: width,
                cropHeight: height
            } = this;
            // 限制区域
            if ((left <= 0 && gapX < 0 && this.action.indexOf('w') > -1)
                || (left + width >= this.width && gapX > 0 && this.action.indexOf('e') > -1)
                || (top <= 0 && gapY < 0 && this.action.indexOf('n') > -1)
                || (top + height >= this.height && gapY > 0 && this.action.indexOf('s') > -1)
            ) {
                return;
            }
            function check() {
                if (width < 0) {
                    width = -width;
                    left -= width;
                    if (this.action.indexOf('w') > -1) {
                        this.action = this.action.replace('w', 'e');
                    }
                    else if (this.action.indexOf('e') > -1) {
                        this.action = this.action.replace('e', 'w');
                    }
                }
                if (height < 0) {
                    height = -height;
                    top -= height;
                    if (this.action.indexOf('n') > -1) {
                        this.action = this.action.replace('n', 's');
                    }
                    else if (this.action.indexOf('s') > -1) {
                        this.action = this.action.replace('s', 'n');
                    }
                }
            }
            function limit() {
                if (left < 0) {
                    width = Math.max(width + left, 0);
                    left = 0;
                    if (this.localRatio !== 0) {
                        top += height - width / this.ratio;
                        height = width / this.ratio;
                    }
                }
                if (left + width > this.width) {
                    width = Math.max(this.width - left, 0);
                    if (this.localRatio !== 0) {
                        height = width / this.ratio;
                    }
                }
                if (top < 0) {
                    height = Math.max(height + top, 0);
                    top = 0;
                    if (this.localRatio !== 0) {
                        left += width - height * this.ratio;
                        width = height * this.ratio;
                    }
                }
                if (top + height > this.height) {
                    height = Math.max(this.height - top, 0);
                    if (this.localRatio !== 0) {
                        width = height * this.ratio;
                    }
                }
            }
            switch (this.action) {
                case 'nw':
                    width -= gapX;
                    left += gapX;
                    if (this.localRatio === 0) {
                        height -= gapY;
                        top += gapY;
                    }
                    else {
                        height = width / this.ratio;
                        top -= height - this.cropHeight;
                    }
                    break;
                case 'n':
                    height -= gapY;
                    top += gapY;
                    if (this.localRatio !== 0) {
                        width = height * this.ratio;
                        left -= (width - this.cropWidth) / 2;
                    }
                    break;
                case 'ne':
                    width += gapX;
                    if (this.localRatio === 0) {
                        height -= gapY;
                        top += gapY;
                    }
                    else {
                        height = width / this.ratio;
                        top -= height - this.cropHeight;
                    }
                    break;
                case 'e':
                    width += gapX;
                    if (this.localRatio !== 0) {
                        height = width / this.ratio;
                        top -= (height - this.cropHeight) / 2;
                    }
                    break;
                case 'se':
                    width += gapX;
                    if (this.localRatio === 0) {
                        height += gapY;
                    }
                    else {
                        height = width / this.ratio;
                    }
                    break;
                case 's':
                    height += gapY;
                    if (this.localRatio !== 0) {
                        width = height * this.ratio;
                        left -= (width - this.cropWidth) / 2;
                    }
                    break;
                case 'sw':
                    width -= gapX;
                    left += gapX;
                    if (this.localRatio === 0) {
                        height += gapY;
                    }
                    else {
                        height = width / this.ratio;
                    }
                    break;
                case 'w':
                    width -= gapX;
                    left += gapX;
                    if (this.localRatio !== 0) {
                        height = width / this.ratio;
                        top -= (height - this.cropHeight) / 2;
                    }
            }
            check.call(this);
            limit.call(this);
            this.cropWidth = width;
            this.cropHeight = height;
            this.translateX = left;
            this.translateY = top;
        },
        cropEnd() {
            this.update();
            this.action = this.pageX = this.pageY = null;
        },
        update() {
            let width = this.cropWidth * this.naturalWidth / this.width;
            let height = this.localRatio === 0
                ? this.cropHeight * this.naturalHeight / this.height
                : width / this.localRatio;
            this.$emit('update', {
                x: this.translateX * this.naturalWidth / this.width,
                y: this.translateY * this.naturalHeight / this.height,
                width,
                height
            });
        }
    }
};
</script>
<style lang="less">
@color: #3998fc;
@rgba: rgba(57, 152, 252, 0.75);

.cropper-line(@direction) {
    .line-style() when (@direction = e), (@direction = w) {
        cursor: ew-resize;
        width: 5px;
        top: 0;
    }
    .line-style() when (@direction = n), (@direction = s) {
        cursor: ns-resize;
        height: 5px;
        left: 0;
    }
    .line-style() when (@direction = e) {
        right: -3px;
    }
    .line-style() when (@direction = w) {
        left: -3px;
    }
    .line-style() when (@direction = n) {
        top: -3px;
    }
    .line-style() when (@direction = s) {
        bottom: -3px;
    }
    .line-style();
}
.cropper-point(@direction) {
    .point-style() when (@direction = e), (@direction = w) {
        cursor: ew-resize;
        height: 10px;
        margin-top: -5px;
        top: 50%;
    }
    .point-style() when (@direction = n), (@direction = s) {
        cursor: ns-resize;
        left: 50%;
        width: 10px;
        margin-left: -5px;
    }
    .point-style() when (@direction = ne), (@direction = sw) {
        cursor: nesw-resize;
        width: 10px;
        &:after {
            cursor: nesw-resize;
            content: '';
            display: block;
            width: 3px;
            height: 10px;
            background-color: @color;
        }
    }
    .point-style() when (@direction = nw), (@direction = se) {
        cursor: nwse-resize;
        width: 10px;
        &:after {
            cursor: nwse-resize;
            content: '';
            display: block;
            width: 3px;
            height: 10px;
            background-color: @color;
        }
    }
    .point-style() when (@direction = e) {
        right: 0;
    }
    .point-style() when (@direction = w) {
        left: 0;
    }
    .point-style() when (@direction = n) {
        top: 0;
    }
    .point-style() when (@direction = s) {
        bottom: 0;
    }
    .point-style() when (@direction = ne) {
        right: 0;
        top: 0;
        &:after {
            margin-left: 7px;
        }
    }
    .point-style() when (@direction = sw) {
        left: 0;
        bottom: 0;
        &:after {
            margin-top: -7px;
        }
    }
    .point-style() when (@direction = nw) {
        left: 0;
        top: 0;
    }
    .point-style() when (@direction = se) {
        right: 0;
        bottom: 0;
        &:after {
            margin-top: -7px;
            margin-left: 7px;
        }
    }
    .point-style();
}

.cropper {
    &-container {
        direction: ltr;
        font-size: 0;
        line-height: 0;
        position: relative;
        user-select: none;
        img {
            display: block;
            width: 100%;
            height: 100%;
        }
    }
    &-wrap-box,
    &-modal-box,
    &-crop-box {
        position: absolute;
        bottom: 0;
        left: 0;
        right: 0;
        top: 0;
        overflow: hidden;
    }
    &-modal-box {
        background-color: #000;
        opacity: .5;
        cursor: crosshair;
    }
    &-view-box {
        display: block;
        width: 100%;
        height: 100%;
        outline: 1px solid @color;
        outline-color: @rgba;
        overflow: hidden;
    }
    &-dashed {
        border: 0 dashed #eee;
        display: block;
        opacity: .5;
        position: absolute;

        &.dashed-h {
            border-bottom-width: 1px;
            border-top-width: 1px;
            height: calc(100% / 3);
            left: 0;
            top: calc(100% / 3);
            width: 100%;
        }

        &.dashed-v {
            border-left-width: 1px;
            border-right-width: 1px;
            height: 100%;
            left: calc(100% / 3);
            top: 0;
            width: calc(100% / 3);
        }
    }
    &-center {
        display: block;
        height: 0;
        left: 50%;
        opacity: .75;
        position: absolute;
        top: 50%;
        width: 0;

        &:before,
        &:after {
            background-color: #eee;
            content: ' ';
            display: block;
            position: absolute;
        }

        &:before {
            height: 1px;
            left: -3px;
            top: 0;
            width: 7px;
        }

        &:after {
            height: 7px;
            left: 0;
            top: -3px;
            width: 1px;
        }
    }
    &-face,
    &-line,
    &-point {
        display: block;
        height: 100%;
        opacity: .1;
        position: absolute;
        width: 100%;
    }

    &-face {
        background-color: #fff;
        left: 0;
        top: 0;
        cursor: move;
    }

    &-line {
        background-color: @color;

        &.line-e {
            .cropper-line(e);
        }

        &.line-n {
            .cropper-line(n);
        }

        &.line-w {
            .cropper-line(w);
        }

        &.line-s {
            .cropper-line(s);
        }
    }

    &-point {
        background-color: @color;
        width: 3px;
        height: 3px;
        opacity: .9;

        &.point-e {
            .cropper-point(e);
        }

        &.point-n {
            .cropper-point(n);
        }

        &.point-w {
            .cropper-point(w);
        }

        &.point-s {
            .cropper-point(s);
        }

        &.point-ne {
            .cropper-point(ne);
        }

        &.point-nw {
            .cropper-point(nw);
        }

        &.point-sw {
            .cropper-point(sw);
        }

        &.point-se {
            .cropper-point(se);
        }
    }
}
</style>
