<template>
    <div :class="tabsCls">
        <div v-if="position == 'bottom'" :class="[contentPrefixCls, animated ? contentPrefixCls + '-animated' : contentPrefixCls + '-no-animated']" :style="contentStyle">
            <slot></slot>
        </div>
        <div role="tablist" :class="prefixCls + '-bar'" tabindex="0">
            <div v-if="(!!$slots && $slots.extra) || (type === 'editable-card' && !hideAdd)" style="float: right;">
                <div class="ant-tabs-extra-content">
                    <slot name="extra"></slot>
                    <span v-if="type === 'editable-card' && !hideAdd" @click="onAdd">
                        <i :class="`anticon anticon-plus ${prefixCls}-new-tab`"></i>
                    </span>
                </div>
            </div>
            <div :class="navContainerCls">
                <span v-if="isScroll" unselectable="unselectable"
                      :class="[tabPrefixCls + '-prev',{[tabPrefixCls + '-btn-disabled']: tabTransform == 0}]"
                      @click="before">
                    <span :class="tabPrefixCls + '-prev-icon'"></span>
                </span>
                <span v-if="isScroll" unselectable="unselectable"
                      :class="[tabPrefixCls + '-next',{[tabPrefixCls + '-btn-disabled']: tabTransform + navScrollWH >= navWH}]"
                      @click="next">
                    <span :class="tabPrefixCls + '-next-icon'"></span>
                </span>
                <div ref="navWrap" :class="navPrefixCls + '-wrap'">
                    <div ref="navScroll" :class="navPrefixCls + '-scroll'">
                        <div ref="nav" :class="[navPrefixCls, animated ? navPrefixCls + '-animated' : navPrefixCls + '-no-animated']"
                             :style="{ transform: 'translate3d(-' + tabTransform + 'px, 0px, 0px)' }">
                            <div :class="[inkBarPrefixCls, animated ? inkBarPrefixCls + '-animated' : inkBarPrefixCls + '-no-animated']" :style="barStyle"></div>
                            <div v-for="(tab, index) in tabs" :key="tab.tabKey" role="tab" v-bind:aria-disabled="tab.disabled" v-bind:aria-selected="index == activeIndex"
                                 :class="[tabPrefixCls, {[tabPrefixCls + '-active']: index == activeIndex, [tabPrefixCls + '-disabled']: tab.disabled}]"
                                 @click="selectTab(index)">
                                <span v-if="tab.icon !== ''" >
                                    <i :class="'anticon anticon-' + tab.icon"></i>
                                    {{ tab.tab }}
                                    <i v-if="type === 'editable-card'" class="anticon anticon-close" @click.stop="onRemove(tab.tabKey)"></i>
                                </span>
                                <template v-else="">
                                    {{ tab.tab }}
                                    <i v-if="type === 'editable-card'" class="anticon anticon-close" @click.stop="onRemove(tab.tabKey)"></i>
                                </template>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <div v-if="position != 'bottom'" :class="[contentPrefixCls, animated ? contentPrefixCls + '-animated' : contentPrefixCls + '-no-animated']" :style="contentStyle">
            <slot></slot>
        </div>
    </div>
</template>

<script lang="babel">
    import emitter from '../../mixins/emitter';

    export default{
        name: 'Tabs',
        mixins: [emitter],
        props: {
            activeTabKey: String,
            type: {
                type: String,
                default: 'line'
            },
            size: String,
            position: {
                type: String,
                default: 'top'
            },
            hideAdd: {
                type: Boolean,
                default: false
            },
            animated: {
                type: Boolean,
                default: true
            }
        },
        model: {
            prop: 'activeTabKey',
            event: 'change',
        },
        data() {
            return {
                prefixCls: 'ant-tabs',
                transitionTime: 300,
                activatedTabKey: this.activeTabKey || 0,
                activeIndex: 0,
                tabs: [],
                tabWH: 0,
                tabMarginRB: 0,
                isScroll: false,
                navWH: 0,
                navScrollWH: 0,
                moveWH: 0,
                tabTransform: 0,
                screenWH: this.getClientWH(document.body),
                preTabPanesCount: 0,
            }
        },
        created() {
            window.addEventListener('resize',()=> {
                this.screenWH = this.getClientWH(document.body);
            });
            this.$on('tabs.disabledItem', (tabPane)=> {
                this.disableTab(tabPane.tabKey, tabPane.disabled);
            });
        },
        mounted() {
            this.updateTabs();
            this.$nextTick(function () {
                /* 直接调用并不会触发视图更新，而在第一次看不到inkBar，所以使用事件排队 */
                this.updateIndicator();
            });
        },
        updated() {
            this.$nextTick(function () {
                let that = this;
                this.updateIndicator();
                /* 当有增加或删除操作时，要更新tabs */
                if (this.tabPanesCount() != this.preTabPanesCount) {
                    this.updateTabs();
                    if (this.scrollToActiveTabThread) {
                        clearTimeout(this.scrollToActiveTabThread)
                    }
                    this.scrollToActiveTabThread = setTimeout(function () {
                        that.scrollToActiveTab();
                    }, that.transitionTime);
                }
            });
        },
        methods: {
            getOffsetWH(element) {
                var prop = 'offsetWidth';
                if (this.isVertical) {
                    prop = 'offsetHeight';
                }
                return element[prop];
            },
            getMarginRB(element) {
                var prop = 'marginRight';
                if (this.isVertical) {
                    prop = 'marginBottom';
                }
                return element[prop];
            },
            getClientWH(element) {
                var prop = 'clientWidth';
                if (this.isVertical) {
                    prop = 'clientHeight';
                }
                return element[prop];
            },
            getOffsetLT(element) {
                var prop = 'left';
                if (this.isVertical) {
                    prop = 'top';
                }
                return element.getBoundingClientRect()[prop];
            },
            tabPanesCount() {
                let count = 0;
                for (let child of this.$children) {
                    if (child.$options.name !== 'TabPane') continue;
                    count++;
                }
                return count;
            },
            updateTabs() {
                let temp_tabs = [];
                let tabPaneIndex = 0;
                for (let child of this.$children) {
                    if (child.$options.name !== 'TabPane') continue;
                    temp_tabs.push({
                        tab: child.tab,
                        icon: child.icon || '',
                        tabKey: child.tabKey || tabPaneIndex,
                        disabled: child.disabled
                    });

                    if (this.activatedTabKey === child.tabKey) {
                        this.activeIndex = tabPaneIndex;
                        this.broadcast('TabPane', 'tabPane.activeTabKey', this.activatedTabKey);
                    }
                    tabPaneIndex++;
                }
                this.$set(this, 'tabs', temp_tabs);
                this.preTabPanesCount = tabPaneIndex;

                if (tabPaneIndex > 0) {
                    if (this.activeIndex >= tabPaneIndex) {
                        /* 防止最后一个被删除后没有内容显示 */
                        this.activeIndex = tabPaneIndex - 1;
                        this.activatedTabKey = this.tabs[this.activeIndex].tabKey;
                        this.broadcast('TabPane', 'tabPane.activeTabKey', this.activatedTabKey);
                    } else if (this.tabs[this.activeIndex].tabKey !== this.activatedTabKey) {
                        /* 删除当前active的tab时，要激活后面一个tab */
                        this.activatedTabKey = this.tabs[this.activeIndex].tabKey;
                        this.broadcast('TabPane', 'tabPane.activeTabKey', this.activatedTabKey);
                    }
                }

                this.$nextTick(function() {
                    this.updateScroll();
                })
            },
            updateIndicator() {
                let tab = this.$el.querySelector('.' + this.tabPrefixCls);
                try {
                    this.tabWH = this.getOffsetWH(tab);
                    this.tabMarginRB = parseInt(this.getMarginRB(getComputedStyle(tab, false)));
                    this.updateScroll();
                } catch (e) {
                    /* Do nothing */
                }
            },
            updateScroll() {
                this.navWH = this.getOffsetWH(this.$refs.nav);
                this.navScrollWH = this.getOffsetWH(this.$refs.navScroll);

                if (this.navScrollWH < this.navWH) {
                    this.isScroll = true;
                } else {
                    this.isScroll = false;
                }

                if (this.tabTransform >= this.navWH) {
                    this.before();
                }
            },
            scrollToActiveTab() {
                var navWrap = this.$refs.navWrap,
                    activeTab = this.getActiveTab();

                if (activeTab) {
                    var activeTabWH = this.getOffsetWH(activeTab);
                    var activeTabMarginRB = parseInt(this.getMarginRB(getComputedStyle(activeTab, false)));
                    var navWrapWH = this.getOffsetWH(navWrap);
                    var navWrapOffset = this.getOffsetLT(navWrap);
                    var activeTabOffset = this.getOffsetLT(activeTab);
                    if (navWrapOffset > activeTabOffset) {
                        this.tabTransform -= navWrapOffset - activeTabOffset;
                    } else if (navWrapOffset + navWrapWH < activeTabOffset + activeTabWH + activeTabMarginRB) {
                        this.tabTransform += (activeTabOffset + activeTabWH + activeTabMarginRB) - (navWrapOffset + navWrapWH);
                    } else {
                        this.scrollToLastTab();
                    }
                }
            },
            scrollToLastTab() {
                var navWrap = this.$refs.navWrap,
                    nav = this.$refs.nav,
                    lastTab = this.getLastTab();

                if (lastTab) {
                    var lastTabbWH = this.getOffsetWH(lastTab);
                    var lastTabbWHMarginRB = parseInt(this.getMarginRB(getComputedStyle(lastTab, false)));
                    var navWrapWH = this.getOffsetWH(navWrap);
                    var navWH = this.navWH = this.getOffsetWH(nav);
                    var navWrapOffset = this.getOffsetLT(navWrap);
                    var lastTabOffset = this.getOffsetLT(lastTab);

                    if (navWrapOffset + navWrapWH > lastTabOffset + lastTabbWH + lastTabbWHMarginRB) {
                        if (navWrapWH < navWH) {
                            this.tabTransform = navWH - navWrapWH;
                        } else {
                            this.tabTransform = 0;
                        }
                    }
                }
            },
            getActiveTab() {
                return this.$el.querySelector('.' + this.tabPrefixCls + '-active');
            },
            getLastTab() {
                let tab = null;
                let tabs = this.$el.querySelectorAll('.' + this.tabPrefixCls);
                if (tabs && tabs.length) {
                    tab = tabs[tabs.length - 1];
                }
                return tab;
            },
            disableTab(tabKey, disabled) {
                for (let i = 0; i < this.tabs.length; i++) {
                    let tab = this.tabs[i];
                    if (tab.tabKey == tabKey) {
                        this.$set(this.tabs[i], "disabled", disabled);
                        break;
                    }
                }
            },
            selectTab(index) {
                this.activeIndex = index;
                this.activatedTabKey = this.tabs[index].tabKey;
                this.$nextTick(function () {
                    this.scrollToActiveTab();
                });
                this.broadcast('TabPane', 'tabPane.activeTabKey', this.activatedTabKey);
                this.$emit('tab-click', this.activatedTabKey);
            },
            setOffset(offset) {
                this.tabTransform = offset;
            },
            before() {
                this.moveWH = Math.floor(this.navScrollWH / ( this.tabWH + 24 )) * ( this.tabWH + 24 );
                if (this.tabTransform - this.moveWH >= 0) {
                    this.tabTransform += -1 * this.moveWH;
                } else {
                    this.tabTransform = 0;
                }
            },
            next() {
                this.navScrollWH = this.getOffsetWH(this.$refs.navScroll);
                this.moveWH = Math.floor(this.navScrollWH / ( this.tabWH + 24 )) * ( this.tabWH + 24 );
                if (this.tabTransform + this.moveWH <= this.navWH - this.navScrollWH) {
                    this.tabTransform += this.moveWH;
                } else if (this.tabTransform <= this.navWH - this.navScrollWH) {
                    this.tabTransform = this.navWH - this.navScrollWH;
                }
            },
            onAdd() {
                this.$emit("add");
            },
            onRemove(tabKey) {
                this.$emit("remove", tabKey);
            },
            updateActiveIndex(activatedTabKey) {
                for (let i = 0; i < this.tabs.length; i++) {
                    if (this.tabs[i].tabKey == activatedTabKey) {
                        this.activeIndex = i;
                        this.selectTab(i);
                        break;
                    }
                }
            },
        },
        computed: {
            isVertical() {
                return this.position === 'left' || this.position === 'right';
            },
            tabPrefixCls() { return this.prefixCls + '-tab' },
            navPrefixCls() { return this.prefixCls + '-nav' },
            inkBarPrefixCls() { return this.prefixCls + '-ink-bar' },
            contentPrefixCls() { return this.prefixCls + '-content' },
            tabsCls() {
                const size = {small: 'mini'}[this.size];
                return [
                    this.prefixCls,
                    `${this.prefixCls}-${this.position}`,
                    `${this.prefixCls}-${this.type}`,
                    {
                        [`${this.prefixCls}-card`]: this.type === 'editable-card',
                        [`${this.prefixCls}-${size}`]: !!size,
                        [`${this.prefixCls}-vertical`]: this.isVertical
                    }
                ]
            },
            navContainerCls() {
                return [
                    `${this.navPrefixCls}-container`,
                    {[`${this.navPrefixCls}-container-scrolling`]: this.isScroll}
                ]
            },
            barStyle() {
                let barStyle =  {
                    transform: 'translate3d(' + (this.tabWH + this.tabMarginRB) * this.activeIndex + 'px, 0px, 0px)'
                };
                if (this.isVertical) {
                    barStyle.height = this.tabWH + 'px';
                    barStyle.transform = 'translate3d(0px, ' + (this.tabWH + this.tabMarginRB) * this.activeIndex + 'px, 0px)';
                } else {
                    barStyle.width = this.tabWH + 'px';
                }
                return barStyle;
            },
            contentStyle() {
                let contentStyle =  {};
                if (!this.isVertical) {
                    contentStyle.marginLeft = -100 * this.activeIndex + '%';
                }
                return contentStyle;
            }
        },
        watch: {
            position(value, oldValue) {
                let that = this;
                if ( value == 'bottom' || oldValue == 'bottom') {
                    this.$nextTick(function () {
                        this.broadcast('TabPane', 'tabPane.activeTabKey', this.activatedTabKey);
                    }, 0);
                }
                setTimeout(function () {
                    that.scrollToActiveTab();
                }, this.transitionTime);
            },
            screenWH() {
                let that = this;
                if (that.resizeThead) {
                    clearTimeout(that.resizeThead);
                }
                that.resizeThead = setTimeout(function () {
                    that.updateScroll();
                }, this.transitionTime)
            },
            activeTabKey(value) {
                this.activatedTabKey = value;
                this.updateActiveIndex(value);
            },
            activatedTabKey(value) {
                this.$emit('change', value);
            },
            activeIndex(value) {
                this.$emit('index-change', value);
            },
        }
    }
</script>
