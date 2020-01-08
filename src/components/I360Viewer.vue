<template>
    <div ref="imageLibrary">
        <div class="header" v-if="show_header">
            <span class="bookTitle">360&deg; Product Viewer</span>
            <span class="title"></span>
        </div>

        <div id="virtualToolboxInsert">

            <div id="viewport-wrapper" ref="viewportWrapper">
                <div class="viewport" ref="viewport">
                    <canvas 
                        class="image-container" 
                        ref="imageContainer" 
                        @mouseup="stopMoving" 
                        @mousedown="startMoving" 
                        @mousemove="doMoving" 
                        @touchstart="touchStart"
                        @touchend="touchEnd"
                        @touchmove="touchMove"
                        @wheel="zoomImage"
                    ></canvas>
                    <div class="screen-toggle"></div>
                </div>
                <div id='drag-icon'></div>
            </div>
            
            <abbr title="Fullscreen Toggle">
                <div class="fullscreen-toggle" @click="toggleFullScreen">
                    <div class="fullscreen-toggle-btn" ref="enterFullScreenIcon">
                        <svg version="1.1" id="Layer_1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px"
                        width="100%" height="100%" viewBox="0 0 512 512" enable-background="new 0 0 512 512" xml:space="preserve">
                            <g>
                                <polygon points="396.795,396.8 320,396.8 320,448 448,448 448,320 396.795,320 	"/>
                                <polygon points="396.8,115.205 396.8,192 448,192 448,64 320,64 320,115.205 	"/>
                                <polygon points="115.205,115.2 192,115.2 192,64 64,64 64,192 115.205,192 	"/>
                                <polygon points="115.2,396.795 115.2,320 64,320 64,448 192,448 192,396.795 	"/>
                            </g>
                        </svg>
                    </div>
                    <div class="fullscreen-toggle-btn" ref="leaveFullScreenIcon">
                        <svg version="1.1" id="Layer_1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" width="30px" height="30px" viewBox="0 0 512 512" enable-background="new 0 0 512 512" xml:space="preserve">
                            <g>
                                <path d="M64,371.2h76.795V448H192V320H64V371.2z M140.795,140.8H64V192h128V64h-51.205V140.8z M320,448h51.2v-76.8H448V320H320
                                    V448z M371.2,140.8V64H320v128h128v-51.2H371.2z"/>
                            </g>
                        </svg>
                    </div>
                </div>
            </abbr>

            <div id="safety" ref="safety"></div>

        </div>

        <div id="menu-btns">
            <div id="navigate-btns">
                <div class="menu-btns" @click="moveUp">
                    <i class="fa fa-chevron-up"></i>
                </div>
                <div class="menu-btns" @click="moveDown">
                    <i class="fa fa-chevron-down"></i>
                </div>
                <div class="menu-btns" @click="resetPosition">
                    <i class="fa fa-redo-alt"></i>
                </div>
                <div class="menu-btns" @click="moveLeft">
                    <i class="fa fa-chevron-left"></i>
                </div>
                <div class="menu-btns" @click="moveRight">
                    <i class="fa fa-chevron-right"></i>
                </div>
            </div>
            <div id="zoom-btns">
                <div class="menu-btns" @click="zoomIn">
                    <i class="fa fa-search-plus"></i>
                </div>
                <div class="menu-btns" @click="zoomOut">
                    <i class="fa fa-search-minus"></i>
                </div>
                <div class="menu-btns" @click="cropImage">
                    <i class="fa fa-download"></i>
                </div>
            </div>
            
            <!-- Controls -->
            <!-- <div class="controls">

                <div class="navigate">
                    <button class="s_up" @click="moveUp"></button>
                    <div>
                        <button class="s_left" @click="moveLeft"></button>
                        <button class="reset" @click="resetPosition"></button>
                        <button class="s_right" @click="moveRight"></button>
                    </div>
                    <button class="s_down" @click="moveDown"></button>
                </div>
                <div class="zoom">
                    <button class="z_in" @click="zoomIn"></button>
                    <button class="z_out" @click="zoomOut"></button>
                </div>
                <button class="crop" @click="cropImage"></button>

            </div> -->
            <!-- /Controls -->
        </div>

    </div>
</template>

<script>

export default {
    name: 'I360Viewer',
    props: {
        images: {
            type: Array,
            require: true,
        },
        imageData: {
            type: Array,
            require: true,
        },
        show_header: {
            type: Boolean,
            require: true,
            default: false,
        },
        spinReverse: {
            type: Boolean,
            require: true,
            default: false,
        },
        amount: {
            type: Number,
            require: true,
            default: 24,
        }
    },
    data(){
        return {
            minScale: 0.5,
            maxScale: 4,
            scale: 0.2,
            customOffset: 10,
            currentScale: 1.0,
            currentTopPosition: 0,
            currentLeftPosition: 0,
            selectMenuOption: 1,
            currentImage: null,
            dragging: false,
            canvas: null,
            ctx: null,
            dragStart: null,
            dragged: false,
            lastX: 0,
            lastY: 0,
            currentCanvasImage: null,
            isFullScreen: false,
            viewPortElementWidth: null,
            movementStart: 0,
            movement: false,
            dragSpeed: 150,
            speedFactor: 13,
            activeImage: 1,
            stopAtEdges: false,
            imagesLoaded: false,
            loadedImages: 0,
        }
    },
    watch: {
        currentImage(value){
            this.setImage()
        },
        currentLeftPosition(value){
            this.redraw()
        },
        currentTopPosition(value){
            this.redraw()
        },
        viewPortElementWidth(value){
            this.update()
        }
    },
    mounted(){
        this.toggleFullScreen()
        this.initData()
    },
    methods: {
        initData(){
            this.loadInitialImage()

            this.canvas = this.$refs.imageContainer
            this.ctx = this.canvas.getContext('2d')
            this.canvas.addEventListener('mouseup', this.stopMoving);
            window.addEventListener('resize', this.resizeWindow);
            this.resizeWindow()
        },
        loadInitialImage(){
            this.changeImage(this.imageData[0])
        },
        resizeWindow(){
            this.setImage()
        },
        zoomIn(evt) {
            this.zoom(2)
        },
        zoomOut(evt) {
            this.zoom(-2)
        },
        moveLeft() {
            this.currentLeftPosition += this.customOffset;
        },
        moveRight() {
            this.currentLeftPosition -= this.customOffset;
        },
        moveUp() {
            this.currentTopPosition += this.customOffset;
        },
        moveDown() {
            this.currentTopPosition -= this.customOffset;
        },
        resetPosition(){
            this.setImage()
        },
        setImage(){
            //this.resetPosition()
            this.currentLeftPosition = this.currentTopPosition = 0
            this.currentCanvasImage = new Image()
            this.currentCanvasImage.crossOrigin='anonymous'
            this.currentCanvasImage.src = this.currentImage
            
            this.currentCanvasImage.onload = () => {
                let viewportElement = this.$refs.viewport.getBoundingClientRect()
                this.canvas.width  = viewportElement.width;
                this.canvas.height = viewportElement.height;
                this.trackTransforms(this.ctx)

                this.redraw()
            }

            this.currentCanvasImage.onerror = () => {
                console.log('cannot load this image')
            }
        },
        redraw(){
            let p1 = this.ctx.transformedPoint(0,0);
            let p2 = this.ctx.transformedPoint(this.canvas.width,this.canvas.height)
            let hRatio = this.canvas.width / this.currentCanvasImage.width
            let vRatio =  this.canvas.height / this.currentCanvasImage.height
            let ratio  = Math.min(hRatio, vRatio);
            let centerShift_x = (this.canvas.width - this.currentCanvasImage.width*ratio )/2
            let centerShift_y = (this.canvas.height - this.currentCanvasImage.height*ratio )/2
            this.ctx.clearRect(p1.x,p1.y,p2.x-p1.x,p2.y-p1.y);
            
            //center image
            this.ctx.drawImage(this.currentCanvasImage, this.currentLeftPosition, this.currentTopPosition, this.currentCanvasImage.width, this.currentCanvasImage.height,
                      centerShift_x,centerShift_y,this.currentCanvasImage.width*ratio, this.currentCanvasImage.height*ratio);  

        },
        cropImage(){
            let croppedCanvas = this.canvas
            let newCanvas = document.createElement('canvas')
            let croppedCtx = newCanvas.getContext('2d')
            newCanvas.width = croppedCanvas.width
            newCanvas.height = croppedCanvas.height
            let newImage = new Image()
            newImage.src = croppedCanvas.toDataURL();
            newImage.onload = () => {
                croppedCtx.drawImage(newImage, 0, 0, newCanvas.width, newCanvas.height);
                let canvasImage = newCanvas.toDataURL();
                let xhr = new XMLHttpRequest();
                xhr.responseType = 'blob';
                xhr.onload = function () {
                    let a = document.createElement('a');
                    a.href = window.URL.createObjectURL(xhr.response);
                    a.download = 'croppedImage.jpg';
                    a.style.display = 'none';
                    document.body.appendChild(a);
                    a.click();
                    a.remove()
                };
                xhr.open('GET', canvasImage);
                xhr.send();
            }
        },
        lpad(str, padString, length){
            str = str.toString();

            while (str.length < length)
                str = padString + str;
            return str;
        },
        changeImage(fig){
            this.currentImage = fig
        },
        showMenu(){
            this.$refs.showMenuIcon.style.display = 'block'
            this.$refs.hideMenuIcon.style.display = 'none'
            this.$refs.menublock.classList.remove('closed');
            this.$refs.viewportWrapper.classList.remove('wide');
            this.$refs.safety.classList.remove('wide');
        },
        hideMenu(){
            this.$refs.showMenuIcon.style.display = 'none'
            this.$refs.hideMenuIcon.style.display = 'block'
            this.$refs.menublock.classList.add('closed');
            this.$refs.viewportWrapper.classList.add('wide');
            this.$refs.safety.classList.add('wide');
        },
        onMove(pageX){
            if (pageX - this.movementStart >= this.speedFactor) {
                let itemsSkippedRight = Math.floor((pageX - this.movementStart) / this.speedFactor) || 1;
                
                this.movementStart = pageX;

                if (this.spinReverse) {
                    this.moveActiveIndexDown(itemsSkippedRight);
                } else {
                    this.moveActiveIndexUp(itemsSkippedRight);
                }

                this.redraw();

            } else if (this.movementStart - pageX >= this.speedFactor) {

                let itemsSkippedLeft = Math.floor((this.movementStart - pageX) / this.speedFactor) || 1;
                
                this.movementStart = pageX;

                if (this.spinReverse) {
                    this.moveActiveIndexUp(itemsSkippedLeft);
                } else {
                    this.moveActiveIndexDown(itemsSkippedLeft);
                }

                this.redraw();
            }
        },
        startMoving(evt){
            this.movement = true
            this.movementStart = event.pageX;
            this.$refs.viewport.style.cursor = 'grabbing';
        },
        doMoving(evt){
            if(this.movement){
                this.onMove(evt.clientX)
            }
        },
        moveActiveIndexUp(itemsSkipped) {

            if (this.stopAtEdges) {
                if (this.activeImage + itemsSkipped >= this.amount) {
                    this.activeImage = this.amount;
                } else {
                    this.activeImage += itemsSkipped;
                }
            } else {
                this.activeImage = (this.activeImage + itemsSkipped) % this.amount || this.amount;
            }
            
            this.update()
        },
        moveActiveIndexDown(itemsSkipped) {

            if (this.stopAtEdges) {
                if (this.activeImage - itemsSkipped <= 1) {
                    this.activeImage = 1;
                } else {
                    this.activeImage -= itemsSkipped;
                }
            } else {
                if (this.activeImage - itemsSkipped < 1) {
                    this.activeImage = this.amount + (this.activeImage - itemsSkipped);
                } else {
                    this.activeImage -= itemsSkipped;
                }
            }
            
            this.update()
        },
        update() {
            const image = this.images[this.activeImage - 1];

            this.currentCanvasImage = image

            this.redraw()
        },
        stopMoving(evt){
            this.movement = false
            this.movementStart = 0
            this.$refs.viewport.style.cursor = 'grab'
        },
        touchStart(evt){
            this.movementStart = evt.touches[0].clientX
        },
        touchMove(evt){
            this.onMove(evt.touches[0].clientX)
        },
        touchEnd(){
            this.movementStart = 0
        },
        startDragging(evt){
            this.dragging = true
            document.body.style.mozUserSelect = document.body.style.webkitUserSelect = document.body.style.userSelect = 'none';
            this.lastX = evt.offsetX || (evt.pageX - this.canvas.offsetLeft);
			this.lastY = evt.offsetY || (evt.pageY - this.canvas.offsetTop);
			this.dragStart = this.ctx.transformedPoint(this.lastX,this.lastY);
			this.dragged = false;
        },
        doDragging(evt){
            this.lastX = evt.offsetX || (evt.pageX - this.canvas.offsetLeft);
			this.lastY = evt.offsetY || (evt.pageY - this.canvas.offsetTop);
            this.dragged = true;

			if (this.dragStart){
				let pt = this.ctx.transformedPoint(this.lastX,this.lastY);
				this.ctx.translate(pt.x-this.dragStart.x,pt.y-this.dragStart.y);
                //redraw();
                this.redraw();
            }
        },
        stopDragging(evt){
            this.dragging = false
            this.dragStart = null
            //zoom on click
            //if (!this.dragged) this.zoom(evt.shiftKey ? -1 : 1 );
        },
        zoom(clicks){
            //console.log(this.lastX + ' - ' + this.lastY)
            let pt = this.ctx.transformedPoint(this.lastX,this.lastY);
			this.ctx.translate(pt.x,pt.y);
            let factor = Math.pow(1.01,clicks);
            //console.log(factor)

            if(factor > 1){
                this.currentScale += factor
            }else{
                this.currentScale -= factor
            }
            
            //console.log(this.currentScale)
			this.ctx.scale(factor,factor);
			this.ctx.translate(-pt.x,-pt.y);
			this.redraw();
        },
        zoomImage(evt){
            this.lastX = evt.offsetX || (evt.pageX - this.canvas.offsetLeft);
            this.lastY = evt.offsetY || (evt.pageY - this.canvas.offsetTop);
            
            var delta = evt.wheelDelta ? evt.wheelDelta/40 : evt.deltaY ? -evt.deltaY : 0;
            
			if (delta) this.zoom(delta);
            return evt.preventDefault() && false;
                
        },
        trackTransforms(ctx){
            var svg = document.createElementNS("http://www.w3.org/2000/svg",'svg');
            var xform = svg.createSVGMatrix();
            this.ctx.getTransform = function(){ return xform; };
            
            var savedTransforms = [];
            var save = ctx.save;
            this.ctx.save = () => {
                savedTransforms.push(xform.translate(0,0));
                return save.call(this.ctx);
            };
            var restore = ctx.restore;
            this.ctx.restore = () => {
                xform = savedTransforms.pop();
                return restore.call(this.ctx);
            };

            var scale = this.ctx.scale;
            this.ctx.scale = (sx,sy) => {
                xform = xform.scaleNonUniform(sx,sy);
                return scale.call(this.ctx,sx,sy);
            };
            var rotate = this.ctx.rotate;
            this.ctx.rotate = (radians) => {
                xform = xform.rotate(radians*180/Math.PI);
                return rotate.call(this.ctx,radians);
            };
            var translate = this.ctx.translate;
            this.ctx.translate = (dx,dy) => {
                xform = xform.translate(dx,dy);
                return translate.call(this.ctx,dx,dy);
            };
            var transform = this.ctx.transform;
            this.ctx.transform = (a,b,c,d,e,f) => {
                var m2 = svg.createSVGMatrix();
                m2.a=a; m2.b=b; m2.c=c; m2.d=d; m2.e=e; m2.f=f;
                xform = xform.multiply(m2);
                return transform.call(this.ctx,a,b,c,d,e,f);
            };
            var setTransform = this.ctx.setTransform;
            this.ctx.setTransform = (a,b,c,d,e,f) => {
                xform.a = a;
                xform.b = b;
                xform.c = c;
                xform.d = d;
                xform.e = e;
                xform.f = f;
                return setTransform.call(this.ctx,a,b,c,d,e,f);
            };
            var pt  = svg.createSVGPoint();
            this.ctx.transformedPoint = (x,y) => {
                pt.x=x; pt.y=y;
                return pt.matrixTransform(xform.inverse());
            }
        },
        toggleFullScreen(){
            this.setImage()
            this.isFullScreen = !this.isFullScreen
            
            if(this.isFullScreen){
                //exit full screen
                this.$refs.imageLibrary.classList.add('main')
                this.$refs.imageLibrary.classList.add('fullscreen')
                this.$refs.enterFullScreenIcon.style.display = 'none'
                this.$refs.leaveFullScreenIcon.style.display = 'block'
            }else{
                //enter full screen
                this.$refs.imageLibrary.classList.remove('main')
                this.$refs.imageLibrary.classList.remove('fullscreen')
                this.$refs.enterFullScreenIcon.style.display = 'block'
                this.$refs.leaveFullScreenIcon.style.display = 'none'
            }
        }
    }
}
</script>

<style>
    .menu-toggle-btn, .fullscreen-toggle-btn{
        background-color: #fff;
    }
    
    @media (min-width: 576px){
        #navigate-btns{
            float: left;
        }
        #zoom-btns{
            float: right;
        }
    }

</style>