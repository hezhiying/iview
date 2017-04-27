<template>
    <div :class="classes" ref="cell">
        <template v-if="renderType === 'group'">
            <Icon :type="isOpen?'arrow-down-b':'arrow-right-b'" v-if="row[column.key]" @click.native="toggleGroup" size="16"></Icon>
        </template>
        <template v-if="renderType === 'index'">{{naturalIndex + 1}}</template>
        <template v-if="renderType === 'selection'">
            <Checkbox :value="checked" @on-change="toggleSelect" :disabled="disabled"></Checkbox>
        </template>
        <template v-if="renderType === 'normal'"><span v-html="row[column.key]"></span></template>
    </div>
</template>
<script>
    import Vue from 'vue';
    import Checkbox from '../checkbox/checkbox.vue';
    import Icon from '../icon/icon.vue';
    import mixins from '../../mixins/emitter'
    export default {
        name: 'TableCell',
        components: { Checkbox },
        mixins:[mixins],
        props: {
            prefixCls: String,
            row: Object,
            column: Object,
            naturalIndex: Number,    // index of rebuildData
            index: Number,           // _index of data
            checked: Boolean,
            disabled: Boolean,
            fixed: {
                type: [Boolean, String],
                default: false
            }
        },
        data () {
            return {
                isOpen:false,
                renderType: '',
                uid: -1,
                context: this.$parent.$parent.currentContext
            };
        },
        computed: {
            classes () {
                return [
                    `${this.prefixCls}-cell`,
                    {
                        [`${this.prefixCls}-hidden`]: !this.fixed && this.column.fixed && (this.column.fixed === 'left' || this.column.fixed === 'right'),
                        [`${this.prefixCls}-cell-ellipsis`]: this.column.ellipsis || false
                    }
                ];
            }
        },
        methods: {
            compile () {
                if (this.column.render) {
                    const $parent = this.context;
                    const $ajaxTable = this.$parent.$parent.$parent;
                    const template = this.column.render(this.row, this.column, this.index);
                    const cell = document.createElement('div');
                    cell.innerHTML = template;

                    this.$el.innerHTML = '';
                    let methods = {};
                    Object.keys($parent).forEach(key => {
                        const func = $parent[key];
                        if (typeof(func) === 'function' && (func.name  === 'boundFn' || func.name === 'n')) {
                            methods[key] = func;
                        }
                    });
                    const res = Vue.compile(cell.outerHTML);
                    // todo 临时解决方案
                    const component = new Vue({
                        render: res.render,
                        staticRenderFns: res.staticRenderFns,
                        methods: methods,
                        data () {
                            return Object.assign({ajaxTableVM:$ajaxTable,contextVM:$parent}, $parent._data);
                        }
                    });
                    component.row = this.row;
                    component.column = this.column;

                    const Cell = component.$mount();
                    this.$refs.cell.appendChild(Cell.$el);
                }
            },
            destroy () {

            },
            toggleSelect () {
                this.$parent.$parent.toggleSelect(this.index);
            },
            toggleGroup(){
            	this.isOpen = !this.isOpen;
				this.$parent.$parent.broadcast('TableBody','on-cell-open',[this.row.group,this.isOpen])
            }
        },
        created () {
            if (this.column.type === 'index') {
                this.renderType = 'index';
            } else if (this.column.type === 'selection') {
                this.renderType = 'selection';
            } else if (this.column.type == 'group'){
                this.renderType = 'group';
                this.$on('on-row-collopse',(collopsed)=>{
                    this.isOpen = !collopsed;
                    this.$parent.$parent.broadcast('TableBody','on-cell-open',[this.row.group,this.isOpen]);
                });
            } else if (this.column.render) {
                this.renderType = 'render';
            } else {
                this.renderType = 'normal';
            }
        },
        mounted () {
            this.$nextTick(() => {
                this.compile();
            });
        },
        beforeDestroy () {
            this.destroy();
        },
        watch: {
            naturalIndex () {
                this.destroy();
                this.compile();
            }
        }
    };
</script>
