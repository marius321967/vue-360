<template>
    <div>
        <!-- 360 Viewer Container -->
        <div class="v360-viewer-container" ref="viewerContainer" :id="identifier">
            
            <!-- 360 Viewer Header -->
            <slot name="header"></slot>
            <!--/ 360 Viewer Header -->
            
            <!-- Percentage Loader -->
            <div class="v360-viewport" v-if="!imagesLoaded">
                <div class="v360-spinner-grow"></div>
                <p class="v360-percentage-text">
                    {{ loadProgress }}
                </p>
            </div>
            <!--/ Percentage Loader -->

            <!-- 360 viewport -->
            <div class="v360-viewport" ref="viewport">
                <canvas 
                    class="v360-image-container" 
                    ref="imageContainer" 
                    @wheel="zoomImage"
                    v-hammer:pinch="onPinch"
                    v-hammer:pinchend="onPinch"
                    v-hammer:pinchout="onPinchOut"
                    v-hammer:pinchin="onPinchIn"
                ></canvas>
                <div class="v360-product-box-shadow" 
                    v-if="boxShadow" @wheel="zoomImage"
                    v-hammer:pinch="onPinch"
                    v-hammer:pinchend="onPinch"
                    v-hammer:pinchout="onPinchOut"
                    v-hammer:pinchin="onPinchIn"
                ></div>
            </div>
            <!--/ 360 viewport -->

            <!-- Fullscreen Button -->
            <abbr title="Fullscreen Toggle">
                <div class="v360-fullscreen-toggle text-center" @click="toggleFullScreen">
                    <div class="v360-fullscreen-toggle-btn" :class="(buttonClass == 'dark') ? 'text-light' : 'text-dark'">
                        <i :class="(!isFullScreen) ? 'fas fa-expand text-lg' : 'fas fa-compress text-lg'"></i>
                    </div>
                </div>
            </abbr>
            <!--/ Fullscreen Button -->

            <!-- Buttons Container -->
            <div id="v360-menu-btns" :class="buttonClass">
                <div class="v360-navigate-btns">
                    <div class="v360-menu-btns" @click="togglePlay" :class="(playing) ? 'v360-btn-active' : ''">
                        <i class="fa fa-play" v-if="!playing"></i>
                        <i class="fa fa-pause" v-else></i>
                    </div>
                    <div class="v360-menu-btns" @click="zoomIn">
                        <i class="fa fa-search-plus"></i>
                    </div>
                    <div class="v360-menu-btns" @click="zoomOut">
                        <i class="fa fa-search-minus"></i>
                    </div>
                    <div class="v360-menu-btns" @click="togglePanMode" :class="(panmode) ? 'v360-btn-active' : ''">
                        <i class="fa fa-hand-paper" v-if="!panmode"></i>
                        <span v-else>360&deg;</span>
                    </div>
                    <div class="v360-menu-btns" @click="prev">
                        <i class="fa fa-chevron-left"></i>
                    </div>
                    <div class="v360-menu-btns" @click="next">
                        <i class="fa fa-chevron-right"></i>
                    </div>
                    <div class="v360-menu-btns" @click="resetPosition">
                        <i class="fa fa-sync"></i>
                    </div>
                </div>
            </div>
            <!--/ Buttons Container -->

        </div>
        <!--/ 360 Viewer Container -->
    </div>
</template>

<script>
const uuidv1 = require('uuid/v1');

export default {
    name: 'I360Viewer',
    props: {
        /**
         * URL to directory with all images.
         */
        imagePath: {
            type: String,
            required: false
        },
        /**
         * Image filename template containing {index}
         */
        fileName: {
            type: String,
            required: false
        },
        /**
         * Zero-padding amount.
         */
        fileNamePadding: {
            type: Number,
            required: false
        },
        /**
         * Number of images for a complete rotation.
         */
        amount: {
            type: Number,
            required: true
        },
        /**
         * Use for multi-layer renders.
         * Array of objects containing same image properties (imagePath, fileName).
         */
        layers: {
            type: Array,
            required: false
        },
        spinReverse: {
            type: Boolean,
            required: false,
            default: false
        },
        autoplay: {
            type: Boolean,
            required: false,
            default: false
        },
        loop: {
            type: Number,
            required: false,
            default: 1
        },
        boxShadow: {
            type: Boolean,
            required: false,
            default: false
        },
        buttonClass: {
            type: String,
            required: false,
            default: 'light'
        },
        hotspots: {
            type: Array,
            required: false,
            default: () => []
        },
        identifier: {
            type: String,
            required: false,
            default: () => uuidv1()
        },
    },
    data() {
        return {
            minScale: 0.5,
            maxScale: 4,
            scale: 0.2,
            customOffset: 10,
            currentScale: 1.0,
            currentTopPosition: 0,
            currentLeftPosition: 0,
            selectMenuOption: 1,
            dragging: false,
            canvas: null,
            ctx: null,
            dragStart: null,
            lastX: 0,
            lastY: 0,
            isFullScreen: false,
            viewPortElementWidth: null,
            movementStart: 0,
            movement: false,
            dragSpeed: 150,
            speedFactor: 13,
            activeImageIndex: 1,
            stopAtEdges: false,
            imagesLoaded: false,
            loadedImages: 0,
            centerX: 0,
            centerY: 0,
            panmode: false,
            isMobile: false,
            currentLoop: 0,
            loopTimeoutId: 0,
            images: [], // HTMLImageElement[][] - images for each layer.
            imageUrls: [], // string[][] - URLs for each layer.
            renderLayers: this.layers,
            playing: false,
        }
    },
    watch: {
        currentLeftPosition(value){
            this.redraw()
        },
        currentTopPosition(value){
            this.redraw()
        },
        viewPortElementWidth(value){
            this.update()
        },
        panmode(value){
            this.attachEvents()
        },
        isFullScreen(value){
            if(!value){
                //exit full screen
                this.$refs.viewerContainer.classList.remove('v360-main')
                this.$refs.viewerContainer.classList.remove('v360-fullscreen')
            }else{
                //enter full screen
                this.$refs.viewerContainer.classList.add('v360-main')
                this.$refs.viewerContainer.classList.add('v360-fullscreen')
                
            }
            this.setImage()
        },
        playing(value) {
            if (value) {
                this.play()
            } else {
                this.stop()
            }
        }
    },
    mounted() {
        if (!this.layers && !this.imagePath) 
            console.error('layers or imagePath/fileName properties are required.');

        // Normalize given properties.
        if (!this.renderLayers) {
            this.renderLayers = [  
                {
                    imagePath: this.imagePath,
                    fileName: this.fileName,
                    fileNamePadding: this.fileNamePadding,
                }
            ];
        }

        this.renderLayers.forEach(layer => {
            this.fetchImages(layer);
        });
        document.addEventListener('fullscreenchange', this.exitHandler);
        document.addEventListener('webkitfullscreenchange', this.exitHandler);
        document.addEventListener('mozfullscreenchange', this.exitHandler);
        document.addEventListener('MSFullscreenChange', this.exitHandler);
    },
    methods: {
        /**
         * Initiate the 360 viewer.
         */
        initData() {
            this.checkMobile()
            this.canvas = this.$refs.imageContainer
            this.ctx = this.canvas.getContext('2d')

            this.attachEvents();
            window.addEventListener('resize', this.resizeWindow);
            this.resizeWindow()
            this.loadInitialImage()

            this.playing = this.autoplay
        },

        /**
         * Build and preload the list of images.
         * Must be called only once during lifecycle.
         * @param {Object} layer
         */
        fetchImages(layer) {
            const layerIndex = this.imageUrls.push([]) - 1;
            for (let i = 1; i <= this.amount; i++) {
                const imageIndex = (layer.fileNamePadding) ? this.lpad(i, "0", layer.fileNamePadding) : i
                const fileName = layer.fileName.replace('{index}', imageIndex);
                const filePath = `${layer.imagePath}/${fileName}`
                this.imageUrls[layerIndex].push(filePath)
            }
            
            this.preloadImages(layerIndex);
        },

        lpad(str, padString, length) {
            str = str.toString()

            while (str.length < length) str = padString + str
            return str
        },

        /**
         * Download all images into this.images from this.imageUrls array.
         * @param {Number} layerIndex Index of this.imageUrls to use.
         */
        preloadImages(layerIndex) {
            try {
                this.imageUrls[layerIndex].forEach(src => {
                    this.addImage(src, layerIndex);
                });
            } catch (error) {
                console.error(`Something went wrong while loading images: ${error.message}`);
            }
        },

        /**
         * Load an Image from the network to the local list (images array).
         * @param {String} src Image URL.
         * @param {Number} layerIndex Layer to push the image.
         */
        addImage(src, layerIndex) {
            if (!this.images[layerIndex])
                this.images.push([]);

            const image = new Image();
            image.src = src;
            image.onload = this.onImageLoad.bind(this);
            image.onerror = this.onImageLoad.bind(this);
            this.images[layerIndex].push(image);
        },
        onImageLoad(event) {
            this.loadedImages += 1;

            if (this.loadedImages === this.totalImages) {
                this.onAllImagesLoaded(event);
            }
        },
        onAllImagesLoaded(e) {
            this.imagesLoaded = true
            this.initData()
        },
        togglePlay() {
            this.playing = !this.playing
        },
        play() {
            this.loopTimeoutId = window.setInterval(() => this.loopImages(), 100);
        },
        onSpin() {
            if (this.playing || this.loopTimeoutId) {
                this.stop();
            }
        },
        stop() {
            if(this.activeImageIndex == 1){
                this.currentLoop = 0
            }
            this.playing = false;
            window.clearTimeout(this.loopTimeoutId);
        },
        loopImages() {
            if(this.activeImageIndex == 1){
                if(this.currentLoop == this.loop){
                    this.stop()
                }
                else{
                    this.currentLoop++
                    
                    this.next()
                }
            }
            else{
                this.next()
            }
        },
        next() {
            (this.spinReverse) ? this.turnLeft() : this.turnRight()
        },
        prev() {
            (this.spinReverse) ? this.turnRight() : this.turnLeft()
        },
        turnLeft(){
            this.moveActiveIndexDown(1);
        },
        turnRight(){
            this.moveActiveIndexUp(1);
        },
        loadImages(){
        },
        checkMobile(){
            this.isMobile = !!('ontouchstart' in window || navigator.msMaxTouchPoints);
        },
        loadInitialImage(){
            this.setImage()
        },
        resizeWindow(){
            this.setImage()
        },
        onPinch(evt){
        },
        onPinchEnd(evt){
            this.tempScale = 0
        },
        onPinchIn(evt){
            //alert('pinchin:' + evt.scale)
            this.zoomOut()
        },
        onPinchOut(evt){
            this.zoomIn()
        },
        attachEvents(){
            if(this.panmode){
                this.bindPanModeEvents()
            }else{
                this.bind360ModeEvents()
            }
        },
        bindPanModeEvents(){
            this.$refs.viewport.removeEventListener('touchend', this.touchEnd);
            this.$refs.viewport.removeEventListener('touchstart', this.touchStart);
            this.$refs.viewport.removeEventListener('touchmove', this.touchMove); 

            this.$refs.viewport.addEventListener('touchend', this.stopDragging);
            this.$refs.viewport.addEventListener('touchstart', this.startDragging);
            this.$refs.viewport.addEventListener('touchmove', this.doDragging); 

            this.$refs.viewport.removeEventListener('mouseup', this.stopMoving);
            this.$refs.viewport.removeEventListener('mousedown', this.startMoving);
            this.$refs.viewport.removeEventListener('mousemove', this.doMoving); 

            this.$refs.viewport.addEventListener('mouseup', this.stopDragging);
            this.$refs.viewport.addEventListener('mousedown', this.startDragging);
            this.$refs.viewport.addEventListener('mousemove', this.doDragging);
        },
        bind360ModeEvents(){
            this.$refs.viewport.removeEventListener('touchend', this.stopDragging);
            this.$refs.viewport.removeEventListener('touchstart', this.startDragging);
            this.$refs.viewport.removeEventListener('touchmove', this.doDragging); 

            this.$refs.viewport.addEventListener('touchend', this.touchEnd);
            this.$refs.viewport.addEventListener('touchstart', this.touchStart);
            this.$refs.viewport.addEventListener('touchmove', this.touchMove); 

            this.$refs.viewport.removeEventListener('mouseup', this.stopDragging);
            this.$refs.viewport.removeEventListener('mousedown', this.startDragging);
            this.$refs.viewport.removeEventListener('mousemove', this.doDragging); 
            
            this.$refs.viewport.addEventListener('mouseup', this.stopMoving);
            this.$refs.viewport.addEventListener('mousedown', this.startMoving);
            this.$refs.viewport.addEventListener('mousemove', this.doMoving);
        },
        togglePanMode(){
            this.panmode = !this.panmode
        },
        zoomIn(evt) {
            this.lastX = this.centerX;
            this.lastY = this.centerY
            this.zoom(2)
        },
        zoomOut(evt) {
            this.lastX = this.centerX;
            this.lastY = this.centerY
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
            this.currentScale = 1
            this.activeImageIndex = 1
            this.setImage(true)
        },
        /**
         * Reset to the first image.
         */
        setImage() {
            this.currentLeftPosition = this.currentTopPosition = 0
            const firstImage = this.currentCanvasImages[0];
            
            let viewportElement = this.$refs.viewport.getBoundingClientRect()
            this.canvas.width  = (this.isFullScreen) ? viewportElement.width : firstImage.width
            this.canvas.height = (this.isFullScreen) ? viewportElement.height : firstImage.height
            this.trackTransforms(this.ctx)

            this.redraw()
        },
        /**
         * Render current image onto the canvas.
         */
        redraw() {
            try {
                const firstImage = this.currentCanvasImages[0];
                let p1 = this.ctx.transformedPoint(0,0);
                let p2 = this.ctx.transformedPoint(this.canvas.width,this.canvas.height)

                let hRatio = this.canvas.width / firstImage.width
                let vRatio =  this.canvas.height / firstImage.height
                let ratio  = Math.min(hRatio, vRatio);
                let centerShift_x = (this.canvas.width - firstImage.width*ratio )/2
                let centerShift_y = (this.canvas.height - firstImage.height*ratio )/2
                this.ctx.clearRect(p1.x,p1.y,p2.x-p1.x,p2.y-p1.y);

                this.centerX = firstImage.width*ratio/2
                this.centerY = firstImage.height*ratio/2
                
                //center image
                this.currentCanvasImages.forEach(image => {
                    this.ctx.drawImage(image, 
                        this.currentLeftPosition, 
                        this.currentTopPosition, 
                        firstImage.width, 
                        firstImage.height,
                        centerShift_x,
                        centerShift_y,
                        firstImage.width*ratio, 
                        firstImage.height*ratio);  
                })

                this.addHotspots()

            }
            catch(e){
                this.trackTransforms(this.ctx)
            }

        },
        addHotspots(){
            this.clearHotspots()

            let currentImageHotspots = this.hotspots.filter(h => h.frame == this.activeImageIndex)

            for(let c in currentImageHotspots){
                let hotspotElement = currentImageHotspots[c]
                
                let hotspotPositionX, hotspotPositionY
                
                if(this.canvas.width > this.$refs.viewport.clientWidth){
                    /* hotspotPositionX = hotspotElement.x * this.$refs.viewport.clientWidth * this.currentScale
                    hotspotPositionY = hotspotElement.y * this.$refs.viewport.clientHeight * this.currentScale */
                    hotspotPositionX = hotspotElement.x * this.$refs.viewport.clientWidth
                    hotspotPositionY = hotspotElement.y * this.$refs.viewport.clientHeight
                }else{
                    hotspotPositionX = hotspotElement.x * this.canvas.width
                    hotspotPositionY = hotspotElement.y * this.canvas.height
                }
                
                let divElement = document.createElement('div')
                let spanElement = document.createElement('span')
                let imgElement = document.createElement('img')
                
                imgElement.className = 'hotspot-icon'
                imgElement.src = hotspotElement.icon
                spanElement.className = 'tooltiptext'
                spanElement.innerHTML = hotspotElement.text
                divElement.className = 'tooltip'
                divElement.style.left = hotspotPositionX + 'px'
                divElement.style.top = hotspotPositionY + 'px'
                divElement.appendChild(imgElement)
                divElement.appendChild(spanElement)

                imgElement.addEventListener('click', (e) => {
                    e.preventDefault()
                    this.selectedHotspot = hotspotElement
                    this.openHotspotForm(true)
                })
                
                this.$refs.viewport.appendChild(divElement)
            }
        },
        clearHotspots(){
            let hotspotButtons = document.getElementById(this.identifier).querySelectorAll('.tooltip')
            
            if(hotspotButtons.length)
                hotspotButtons.forEach(element => element.remove())
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
                if (this.activeImageIndex + itemsSkipped >= this.amount) {
                    this.activeImageIndex = this.amount;
                } else {
                    this.activeImageIndex += itemsSkipped;
                }
            } else {
                this.activeImageIndex = (this.activeImageIndex + itemsSkipped) % this.amount || this.amount;
            }
            
            this.update()
        },
        moveActiveIndexDown(itemsSkipped) {

            if (this.stopAtEdges) {
                if (this.activeImageIndex - itemsSkipped <= 1) {
                    this.activeImageIndex = 1;
                } else {
                    this.activeImageIndex -= itemsSkipped;
                }
            } else {
                if (this.activeImageIndex - itemsSkipped < 1) {
                    this.activeImageIndex = this.amount + (this.activeImageIndex - itemsSkipped);
                } else {
                    this.activeImageIndex -= itemsSkipped;
                }
            }
            
            this.update()
        },
        /**
         * Update image pointer based on active image index and redraw().
         */
        update() {
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

            if(this.isMobile){
                this.lastX = evt.touches[0].offsetX || (evt.touches[0].pageX - this.canvas.offsetLeft);
			    this.lastY = evt.touches[0].offsetY || (evt.touches[0].pageY - this.canvas.offsetTop);
            }else{
                this.lastX = evt.offsetX || (evt.pageX - this.canvas.offsetLeft);
			    this.lastY = evt.offsetY || (evt.pageY - this.canvas.offsetTop);
            }
            
			this.dragStart = this.ctx.transformedPoint(this.lastX,this.lastY);
        },
        doDragging(evt){
            
            if(this.isMobile){
                this.lastX = evt.touches[0].offsetX || (evt.touches[0].pageX - this.canvas.offsetLeft);
			    this.lastY = evt.touches[0].offsetY || (evt.touches[0].pageY - this.canvas.offsetTop);
            }else{
                this.lastX = evt.offsetX || (evt.pageX - this.canvas.offsetLeft);
			    this.lastY = evt.offsetY || (evt.pageY - this.canvas.offsetTop);
            }
            
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
        },
        restrictScale(){
            let scale = this.currentScale
            if (scale < this.minScale) {
                scale = this.minScale;
            } else if (scale > this.maxScale) {
                scale = this.maxScale;
            }
            return scale;
        },
        zoom(clicks){
            let factor = Math.pow(1.01,clicks);

            if(factor > 1){
                this.currentScale += factor
            }else{
                if(this.currentScale-factor > 1)
                    this.currentScale -= factor
                else
                    this.currentScale = 1
            }
            
            if(this.currentScale > 1){
                let pt = this.ctx.transformedPoint(this.lastX,this.lastY);
                this.ctx.translate(pt.x,pt.y);
                
                this.ctx.scale(factor,factor);
                this.ctx.translate(-pt.x,-pt.y);
                this.redraw();
            }
        },
        zoomImage(evt){
            this.lastX = evt.offsetX || (evt.pageX - this.canvas.offsetLeft);
            this.lastY = evt.offsetY || (evt.pageY - this.canvas.offsetTop);
            
            var delta = evt.wheelDelta ? evt.wheelDelta/40 : evt.deltaY ? -evt.deltaY : 0;
            
			if (delta) this.zoom(delta);
            return evt.preventDefault() && false;
                
        },
        trackTransforms(ctx){

            return new Promise(resolve => {
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

                resolve(this.ctx)
            })
            
        },
        toggleFullScreen(){
            this.isFullScreen = !this.isFullScreen
        },
    },

    computed: {
        currentImageUrls() {
            return this.imageUrls.map(layerUrls => (layerUrls[ this.activeImageIndex-1 ]));
        },
        currentCanvasImages() {
            return this.images.map(layerImages => (layerImages[ this.activeImageIndex-1 ]));
        },
        loadProgress() {
            return Math.round(this.loadedImages / this.totalImages * 100) + '%';
        },
        totalImages() {
            return this.amount * this.layers.length;
        }
    }
}
</script>