<template>
    <div :class="classes">
        <div :class="[prefixCls + '-bar']">
            <div :class="[prefixCls + '-nav-container']">
                <div :class="[prefixCls + '-nav-wrap']">
                    <div :class="[prefixCls + '-nav-scroll']">
                        <div :class="[prefixCls + '-nav']" v-el:nav>
                            <div :class="barClasses" :style="barStyle"></div>
                            <div :class="tabCls(item)" v-for="item in navList" @click="handleChange($index)">
                                <Icon v-if="item.icon !== ''" :type="item.icon"></Icon>
                                {{ item.label }}
                                <Icon v-if="closable && type === 'card'" type="ios-close-empty" @click.stop="handleRemove($index)"></Icon>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <div :class="contentClasses" :style="contentStyle"><slot></slot></div>
    </div>
</template>
<script>
    import Icon from '../icon/icon.vue';
    import { oneOf, getStyle } from '../../utils/assist';

    const prefixCls = 'ivu-tabs';

    export default {
        components: { Icon },
        props: {
            activeKey: {
                type: [String, Number]
            },
            type: {
                validator (value) {
                    return oneOf(value, ['line', 'card']);
                },
                default: 'line'
            },
            size: {
                validator (value) {
                    return oneOf(value, ['small', 'default']);
                },
                default: 'default'
            },
            animated: {
                type: Boolean,
                default: true
            },
            closable: {
                type: Boolean,
                default: false
            }
        },
        data () {
            return {
                prefixCls: prefixCls,
                navList: [],
                barWidth: 0,
                barOffset: 0
            }
        },
        computed: {
            classes () {
                return [
                    `${prefixCls}`,
                    {
                        [`${prefixCls}-card`]: this.type === 'card',
                        [`${prefixCls}-mini`]: this.size === 'small' && this.type === 'line',
                        [`${prefixCls}-no-animation`]: !this.animated
                    }
                ]
            },
            contentClasses () {
                return [
                    `${prefixCls}-content`,
                    {
                        [`${prefixCls}-content-animated`]: this.animated
                    }
                ]
            },
            barClasses () {
                return [
                    `${prefixCls}-ink-bar`,
                    {
                        [`${prefixCls}-ink-bar-animated`]: this.animated
                    }
                ]
            },
            contentStyle () {
                const x = this.navList.findIndex((nav, index) => nav.key === this.activeKey);
                const p = x === 0 ? '0%' : `-${x}00%`;

                let style = {};
                if (x > -1) {
                    style = {
                        transform: `translateX(${p}) translateZ(0px)`
                    }
                }
                return style;
            },
            barStyle () {
                let style = {
                    display: 'none',
                    width: `${this.barWidth}px`
                };
                if (this.type === 'line') style.display = 'block';
                if (this.animated) {
                    style.transform = `translate3d(${this.barOffset}px, 0px, 0px)`;
                } else {
                    style.left = `${this.barOffset}px`;
                }

                return style;
            }
        },
        methods: {
            getTabs () {
                return this.$children.filter(item => item.$options.name === 'TabPane');
            },
            updateNav () {
                this.navList = [];
                this.getTabs().forEach((pane, index) => {
                    this.navList.push({
                        label: pane.label,
                        icon: pane.icon || '',
                        key: pane.key || index,
                        disabled: pane.disabled
                    });
                    if (!pane.key) pane.key = index;
                    if (index === 0) {
                        if (!this.activeKey) this.activeKey = pane.key || index;
                    }
                });
                this.updateStatus();
                this.updateBar();
            },
            updateBar () {
                this.$nextTick(() => {
                    const index = this.navList.findIndex((nav, index) => nav.key === this.activeKey);
                    const prevTabs = this.$els.nav.querySelectorAll(`.${prefixCls}-tab`);
                    const tab = prevTabs[index];
                    this.barWidth = parseFloat(getStyle(tab, 'width'));

                    if (index > 0) {
                        let offset = 0;
                        const gutter = this.size === 'small' ? 0 : 16;
                        for (let i = 0; i < index; i++) {
                            offset += parseFloat(getStyle(prevTabs[i], 'width')) + gutter;
                        }

                        this.barOffset = offset;
                    } else {
                        this.barOffset = 0;
                    }
                });
            },
            updateStatus () {
                const tabs = this.getTabs();
                tabs.forEach(tab => tab.show = (tab.key === this.activeKey) || this.animated);
            },
            tabCls (item) {
                return [
                    `${prefixCls}-tab`,
                    {
                        [`${prefixCls}-tab-disabled`]: item.disabled,
                        [`${prefixCls}-tab-active`]: item.key === this.activeKey
                    }
                ]
            },
            handleChange (index) {
                const nav = this.navList[index];
                if (nav.disabled) return;
                this.activeKey = nav.key;
                this.updateStatus();
                this.$emit('on-click', nav.key);
            },
            handleRemove (index) {
                const tabs = this.getTabs();
                const tab = tabs[index];
                tab.$destroy(true);

                if (tab.key === this.activeKey) {
                    const newTabs = this.getTabs();
                    let activeKey = -1;

                    if (newTabs.length) {
                        const leftNoDisabledTabs = tabs.filter((item, itemIndex) => !item.disabled && itemIndex < index);
                        const rightNoDisabledTabs = tabs.filter((item, itemIndex) => !item.disabled && itemIndex > index);

                        if (rightNoDisabledTabs.length) {
                            activeKey = rightNoDisabledTabs[0].key;
                        } else if (leftNoDisabledTabs.length) {
                            activeKey = leftNoDisabledTabs[leftNoDisabledTabs.length - 1].key;
                        } else {
                            activeKey = newTabs[0].key;
                        }
                    }
                    this.activeKey = activeKey;
                }
                this.$emit('on-tab-remove', tab.key);
                this.updateNav();
            }
        },
        compiled () {
            this.updateNav();
        },
        watch: {
            activeKey () {
                this.updateBar();
            }
        }
    }
</script>