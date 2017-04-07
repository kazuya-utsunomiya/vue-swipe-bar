<template>
  <div class="swiper"
       :class="[{'dragging': dragging}]"
       @touchstart="_onTouchStart"
       @mousedown="_onTouchStart"
       @wheel="_onWheel">
    <div class="infinity-swipe-wrapper"
         ref="infinitySwipeWrapper"
         :style="styleObject"
         @transitionend="_onTransitionEnd">
      <slot></slot>
    </div>
  </div>
</template>

<script>
  export default {
    name: 'infinitySwipe',
    props: {
      currentPage:{
        type: Number,
        default: 1
      },
      mousewheelControl: {
        type: Boolean,
        default: true
      },
      performanceMode: {
        type: Boolean,
        default: false
      },
      loop: {
        type: Boolean,
        default: false
      },
      speed: {
        type: Number,
        default: 500
      }
    },
    data(){
      return {
        lastPage: 1,
        translateX: 0,
        startTranslate: 0,
        delta: 0,
        dragging: false,
        startPos: null,
        transitioning: false,
        slideEls: [],
        translateOffset: 0,
        transitionDuration: 0
      }
    },
    computed:{
      styleObject: function(){
        return {transform: 'translate3d(' + this.translateX + 'px, 0, 0)', transitionDuration: this.transitionDuration + 'ms'}
      },
      centerOffset: function(){
        let bullet_width = 110;
        if (this.$refs.infinitySwipeWrapper.children[0]){
          bullet_width = this.$refs.infinitySwipeWrapper.children[0].clientWidth;
        }
        return Math.round(this.$el.clientWidth / 2 - (bullet_width / 2));
      }
    },
    mounted(){
      this.$nextTick(function () {
        this.translateOffset = this.centerOffset;
        this._onTouchMove = this._onTouchMove.bind(this);
        this._onTouchEnd = this._onTouchEnd.bind(this);
        this.slideEls = [].map.call(this.$refs.infinitySwipeWrapper.children, el => el);
        if (this.loop) {
          this.$nextTick(function () {
            this._createLoop();
            this.setPage(this.currentPage, true);
          });
        } else {
          this.setPage(this.currentPage);
        }
      });
    },
    watch: {
      currentPage: function(page){
        this.translateOffset = this.centerOffset;
        this.slideEls = [].map.call(this.$refs.infinitySwipeWrapper.children, el => el); // debug
        this.setPage(page, false);
      }
    },
    methods: {
      next() {
        let page = this.currentPage;
        if (page < this.slideEls.length || this.loop) {
          this.setPage(page + 1);
        } else {
          this._revert();
        }
      },
      prev() {
        let page = this.currentPage;
        if (page > 1 || this.loop) {
          this.setPage(page - 1);
        } else {
          this._revert();
        }
      },
      setPage(page, noAnimation) {
//        if (('disabled' in this.buttons[page-1]) && this.buttons[page-1].disabled) return;

//        if (page === 0) {
//          this.currentPage = this.slideEls.length;
//        } else if (page === this.slideEls.length + 1) {
//          this.currentPage = 1;
//        } else {
//          this.currentPage = page;
//        }
        if (!this.dragging){
          let self = this;
          if (this.loop) {
            if (this.delta === 0) {
              this._setTranslate(self._getTranslateOfPage(this.lastPage));
            }
            setTimeout(function () {
              self._setTranslate(self._getTranslateOfPage(page));
              if (noAnimation) return;
              self._onTransitionStart();
            }, 0);
          } else {
            this._setTranslate(this._getTranslateOfPage(page));
            if (noAnimation) return;
            this._onTransitionStart();
          }
        }

        if (this._isPageChanged()) this.$emit('slide-change-page', page);
        this.lastPage = this.currentPage;
      },
      _onTouchStart(e) {
        this.startPos = this._getTouchPos(e);
        this.delta = 0;
        this.startTranslate = this._getTranslateOfPage(this.currentPage);
        this.startTime = new Date().getTime();
        this.dragging = true;
        this.transitionDuration = 0;
        document.addEventListener('touchmove', this._onTouchMove, false);
        document.addEventListener('touchend', this._onTouchEnd, false);
        document.addEventListener('mousemove', this._onTouchMove, false);
        document.addEventListener('mouseup', this._onTouchEnd, false);
      },
      _onTouchMove(e) {
        this.delta = this._getTouchPos(e) - this.startPos;
        if (!this.performanceMode) {
          this._setTranslate(this.startTranslate + this.delta);
          this.$emit('slider-move', this._getTranslate() - this.translateOffset, this.delta);
        }
        if (Math.abs(this.delta) > 0) {
          e.preventDefault();
        }
      },
      _onTouchEnd(e) {
        this.dragging = false;
        this.transitionDuration = this.speed;
//        let isQuickAction = new Date().getTime() - this.startTime < 1000;
//        if (this.delta < -112 || (isQuickAction && this.delta < -15)) {
//          this.next();
//        } else if (this.delta > 112 || (isQuickAction && this.delta > 15)) {
//          this.prev();
//        } else {
//          this._revert();
//        }
        this.setPage(this.currentPage);

        document.removeEventListener('touchmove', this._onTouchMove);
        document.removeEventListener('touchend', this._onTouchEnd);
        document.removeEventListener('mousemove', this._onTouchMove);
        document.removeEventListener('mouseup', this._onTouchEnd);
        this.$emit('touch-end', this.currentPage);
      },
      _onWheel(e) {
        if (this.mousewheelControl) {
          // TODO Support apple magic mouse and trackpad.
          if (!this.transitioning) {
            if (e.deltaY > 0) {
              this.next();
            } else {
              this.prev();
            }
          }
          if (this._isPageChanged()) e.preventDefault();
        }
      },
      _revert() {
        this.setPage(this.currentPage);
      },
      _getTouchPos(e) {
        let key = 'pageX';
        return e.changedTouches ? e.changedTouches[0][key] : e[key];
      },
      _onTransitionStart() {
        this.transitioning = true;
        this.transitionDuration = this.speed;
        if (this._isPageChanged()) {
          this.$emit('slide-change-start', this.currentPage);
        } else {
          this.$emit('slide-revert-start', this.currentPage);
        }
      },
      _onTransitionEnd() {
        this.transitioning = false;
        this.transitionDuration = 0;
        this.delta = 0;
        if (this._isPageChanged()) {
          this.$emit('slide-change-end', this.currentPage);
        } else {
          this.$emit('slide-revert-end', this.currentPage);
        }
      },
      _isPageChanged() {
        return this.lastPage !== this.currentPage;
      },
      _setTranslate(value) {
        let translateName = 'translateX';
        this[translateName] = value;
      },
      _getTranslate(){
        let translateName = 'translateX';
        return this[translateName];
      },
      _getTranslateOfPage(page) {
        if (page === 0) return 0;
        let propName = 'clientWidth';
        return -[].reduce.call(this.slideEls, function (total, el, i) {
            return i > page - 2 ? total : total + el[propName];
          }, 0) + this.translateOffset;
      },
      _createLoop() {
//        this.swipe.buttons = this.swipe.buttons.concat(this.swipe.buttons);
      }
    },
    serverCacheKey: props => props.show
  }
</script>

<style>
  .infinity-swipe {
    margin: 0 auto;
    position: relative;
    overflow: hidden;
    z-index: 1;
  }
  .infinity-swipe-wrapper {
    position: relative;
    width: 100%;
    height: 100%;
    z-index: 1;
    display: flex;
    -webkit-transition-property: -webkit-transform;
    transition-property: transform;
    box-sizing: content-box;
  }
  .infinity-swipe-item {
    flex-shrink: 0;
    height: 100%;
  }
</style>