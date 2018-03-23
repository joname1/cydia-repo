title: 基于Vue2写的上拉加载更多
date: 2017-10-26 11:23:54 +0800
update: 2017-10-26 12:00:00 +0800
author: me
tags:
    - Vue
preview: 世界上本有坑, 踩得人多了就习惯了。

---
代码如下:
```js
<template>
    <div class="loadmore">
        <slot></slot>
        <slot name="bottom">
        </slot>
    </div>
</template>

<style>
    .loadmore{
        width:100%;
    }
</style>

<script>
    export default {
        name: 'loadmore',
        props: {
            maxDistance: {
                type: Number,
                default: 0
            },
            autoFill: {
                type: Boolean,
                default: true
            },
            distanceIndex: {
                type: Number,
                default: 2
            },
            bottomPullText: {
                type: String,
                default: '上拉刷新'
            },
            bottomDropText: {
                type: String,
                default: '释放更新'
            },
            bottomLoadingText: {
                type: String,
                default: '加载中...'
            },
            bottomDistance: {
                type: Number,
                default: 70
            },
            bottomMethod: {
                type: Function
            },
            bottomAllLoaded: {
                type: Boolean,
                default: false
            },
        },
        data() {
            return {
                // 最下面出现的div的位移
                translate: 0,
                // 选择滚动事件的监听对象
                scrollEventTarget: null,
                containerFilled: false,
                bottomText: '',
                // class类名
                bottomDropped: false,
                // 获取监听滚动元素的scrollTop
                bottomReached: false,
                // 滑动的方向   down---向下互动；up---向上滑动
                direction: '',
                startY: 0,
                startScrollTop: 0,
                // 实时的clientY位置
                currentY: 0,
                topStatus: '',
                //  上拉加载的状态    ''     pull: 上拉中
                bottomStatus: '',
            };
        },
        watch: {
            // 改变当前加载在状态
            bottomStatus(val) {
                this.$emit('bottom-status-change', val);
                switch (val) {
                    case 'pull':
                        this.bottomText = this.bottomPullText;
                        break;
                    case 'drop':
                        this.bottomText = this.bottomDropText;
                        break;
                    case 'loading':
                        this.bottomText = this.bottomLoadingText;
                        break;
                }
            }
        },
        methods: {
            onBottomLoaded() {
                this.bottomStatus = 'pull';
                this.bottomDropped = false;
                this.$nextTick(() => {
                    if (this.scrollEventTarget === window) {
                    document.body.scrollTop += 50;
                } else {
                    this.scrollEventTarget.scrollTop += 50;
                }
                this.translate = 0;
            });
                // 注释
                if (!this.bottomAllLoaded && !this.containerFilled) {
                    this.fillContainer();
                }
            },

            getScrollEventTarget(element) {
                let currentNode = element;
                while (currentNode && currentNode.tagName !== 'HTML' &&
                currentNode.tagName !== 'BODY' && currentNode.nodeType === 1) {
                    let overflowY = document.defaultView.getComputedStyle(currentNode).overflowY;
                    if (overflowY === 'scroll' || overflowY === 'auto') {
                        return currentNode;
                    }
                    currentNode = currentNode.parentNode;
                }
                return window;
            },
            //  获取scrollTop
            getScrollTop(element) {
                if (element === window) {
                    return Math.max(window.pageYOffset || 0, document.documentElement.scrollTop);
                } else {
                    return element.scrollTop;
                }
            },
            bindTouchEvents() {
                this.$el.addEventListener('touchstart', this.handleTouchStart);
                this.$el.addEventListener('touchmove', this.handleTouchMove);
                this.$el.addEventListener('touchend', this.handleTouchEnd);
            },
            init() {
                this.bottomStatus = 'pull';
                // 选择滚动事件的监听对象
                this.scrollEventTarget = this.getScrollEventTarget(this.$el);
                if (typeof this.bottomMethod === 'function') {
                    // autoFill 属性的实现   注释
                    this.fillContainer();
                    // 绑定滑动事件
                    this.bindTouchEvents();
                }
            },
            //  autoFill 属性的实现   注释
            fillContainer() {
                if (this.autoFill) {
                    this.$nextTick(() => {
                        if (this.scrollEventTarget === window) {
                        this.containerFilled = this.$el.getBoundingClientRect().bottom >=
                                document.documentElement.getBoundingClientRect().bottom;
                    } else {
                        this.containerFilled = this.$el.getBoundingClientRect().bottom >=
                                this.scrollEventTarget.getBoundingClientRect().bottom;
                    }
                    if (!this.containerFilled) {
                        this.bottomStatus = 'loading';
                        this.bottomMethod();
                    }
                });
                }
            },
            //  获取监听滚动元素的scrollTop
            checkBottomReached() {
                if (this.scrollEventTarget === window) {
                    return document.body.scrollTop + document.documentElement.clientHeight >= document.body.scrollHeight;
                } else {
                    // getBoundingClientRect用于获得页面中某个元素的左，上，右和下分别相对浏览器视窗的位置。 right是指元素右边界距窗口最左边的距离，bottom是指元素下边界距窗口最上面的距离。
                    return this.$el.getBoundingClientRect().bottom <= this.scrollEventTarget.getBoundingClientRect().bottom + 1;
                }
            },
            // ontouchstart 事件
            handleTouchStart(event) {
                // 获取起点的y坐标
                this.startY = event.touches[0].clientY;
                this.startScrollTop = this.getScrollTop(this.scrollEventTarget);
                this.bottomReached = false;
                if (this.bottomStatus !== 'loading') {
                    this.bottomStatus = 'pull';
                    this.bottomDropped = false;
                }
            },
            // ontouchmove事件
            handleTouchMove(event) {
                if (this.startY < this.$el.getBoundingClientRect().top && this.startY > this.$el.getBoundingClientRect().bottom) {
                    // 没有在需要滚动的范围内滚动，不再监听scroll
                    return;
                }
                // 实时的clientY位置
                this.currentY = event.touches[0].clientY;
                //  distance 移动位置和开始位置的差值        distanceIndex---
                let distance = (this.currentY - this.startY) / this.distanceIndex;
                // 根据 distance 判断滑动的方向  并赋予变量   direction  down---向下互动；up---向上滑动
                this.direction = distance > 0 ? 'down' : 'up';
                if (this.direction === 'up') {
                    // 获取监听滚动元素的scrollTop
                    this.bottomReached = this.bottomReached || this.checkBottomReached();
                }
                if (typeof this.bottomMethod === 'function' && this.direction === 'up' &&
                        this.bottomReached && this.bottomStatus !== 'loading' && !this.bottomAllLoaded) {
                    // 有加载函数，是向上拉，有滚动距离，不是正在加载ajax，没有加载到最后一页
                    event.preventDefault();
                    event.stopPropagation();
                    if (this.maxDistance > 0) {
                        this.translate = Math.abs(distance) <= this.maxDistance
                                ? this.getScrollTop(this.scrollEventTarget) - this.startScrollTop + distance : this.translate;
                    } else {
                        this.translate = this.getScrollTop(this.scrollEventTarget) - this.startScrollTop + distance;
                    }
                    if (this.translate > 0) {
                        this.translate = 0;
                    }
                    this.bottomStatus = -this.translate >= this.bottomDistance ? 'drop' : 'pull';
                }
            },
            // ontouchend事件
            handleTouchEnd() {
                if (this.direction === 'up' && this.bottomReached && this.translate < 0) {
                    this.bottomDropped = true;
                    this.bottomReached = false;
                    if (this.bottomStatus === 'drop') {
                        this.translate = '-50';
                        this.bottomStatus = 'loading';
                        this.bottomMethod();
                    } else {
                        this.translate = '0';
                        this.bottomStatus = 'pull';
                    }
                }
                this.direction = '';
            }
        },
        mounted() {
            this.init();
        }
    };
</script>
```
建议抽出来放到common目录下，然后哪个页面需要，就在哪个页面import即可, 然后在引入他的页面写法如下：
```js
<template>
  <section class="finan">
    <!-- 上拉加载更多 -->
    <load-more
    :bottom-method="loadBottom"
    :bottom-all-loaded="allLoaded"
    :bottomPullText='bottomText'
    :auto-fill="false"
    @bottom-status-change="handleBottomChange"
    ref="loadmore">
        <div>
	  这里写你需要的另外的模块
        </div>
        <div v-show="loading" slot="bottom" class="loading"> 这个div是为让上拉加载的时候显示一张加载的gif图
          <img src="./../../assets/main/uploading.gif">
        </div>
    </load-more>
  </section>
</template>
```
然后在data里和methods设置如下：
```js
export default {
        name: 'FinancialGroup',
        props:{
		
        },
        data () {
            return {
                //  上拉加载数据
                scrollHeight: 0,
                scrollTop: 0,
                containerHeight: 0,
                loading: false,
                allLoaded: false,
                bottomText: '上拉加载更多...',
                bottomStatus: '',
                pageNo: 1,
                totalCount: '',
            }
        },
        methods: {
        /* 下拉加载 */
        _scroll: function(ev) {
            ev = ev || event;
            this.scrollHeight = this.$refs.innerScroll.scrollHeight;
            this.scrollTop = this.$refs.innerScroll.scrollTop;
            this.containerHeight = this.$refs.innerScroll.offsetHeight;
        },
        loadBottom: function() {
            this.loading = true;
            this.pageNo += 1;   // 每次更迭加载的页数
            if (this.pageNo == this.totalGetCount) {
                // 当allLoaded = true时上拉加载停止
                this.loading = false;
                this.allLoaded = true;
            }
            api.commonApi(后台接口，请求参数) 这个api是封装的axios有不懂的可以看vue2+vuex+axios那篇文章
                    .then(res => {
                setTimeout(() => {
 	          要使用的后台返回的数据写在setTimeout里面
                  this.$nextTick(() => {
                    this.loading = false;
            	  })
            }, 1000)
         });
        },
        handleBottomChange(status) {
            this.bottomStatus = status;
        },
    }
```