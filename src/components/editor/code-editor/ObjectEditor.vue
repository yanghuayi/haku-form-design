<template>
    <div class="code-editor" ref="editor" :style="{ height: height }"></div>
</template>
 
<script>
    import * as monaco from 'monaco-editor/esm/vs/editor/editor.api';

    export default {
        name: 'ObjectEditor',
        props: {
            value: {
                type: Array
            },
            marker: {
                type: Function
            },
            height: {
                type: String,
                default: '20px'
            },
            options: {
                type: Object,
                default: () => ({})
            },
            /** 是否为表单表达式 */
            isFormExpression: {
                type: Boolean,
                default: false
            },
            unseenLines: {
                type: Array
            },
            placeholder: {
                type: String,
                default: '请输入JSON字符串'
            },
            /** 表单变量 */
            variables: {
                type: Array,
                default: () => []
            }
        },
        inject: ['bus'],
        data: () => ({
            editor: null,
            content: '',
            defaultOptions: {
                wordWrap: 'off', //控制如何换行
                automaticLayout: true,
                fontFamily: "'Fira Code', Consolas, 'Courier New', monospace, 'Microsoft YaHei'"
            },
            /** 停止绘制 */
            __preventTriggerChangeEvent: false
        }),
        watch: {
            value(newVal) {
                this.handerCodeChange(newVal);
            }
        },
        created() {
            monaco.editor.defineTheme('gj-dark', {
                base: 'vs-dark',
                inherit: true,
                rules: [
                    { token: 'custom-variable', foreground: 'ffa500', fontStyle: 'underline' }
                ]
            });

            // validation settings
            monaco.languages.typescript.javascriptDefaults.setDiagnosticsOptions({
                noSemanticValidation: true,
                noSyntaxValidation: false
            });

            // compiler options
            monaco.languages.typescript.javascriptDefaults.setCompilerOptions({
                target: monaco.languages.typescript.ScriptTarget.ESNext,
                allowNonTsExtensions: true
            });

            // 引入变量
            let _variables = this.variables.map(i => `
                ${i.remark ? '/** ' +  i.remark + ' */\n' : ''}${i.keyword} ${i.name}: ${i.type};
            `);
            monaco.languages.typescript.javascriptDefaults.addExtraLib(_variables.join('\n'), 'vars.d.ts');
        },
        mounted() {
            this.initialize();
        },
        beforeDestroy() {
            this.destroy();
        },
        methods: {
            initialize() {
                // onDidChangeOptions 配置改变事件
                // onDidChangeLanguage 语言改变事件

                let _options = {
                    ...this.defaultOptions,
                    ...this.options,
                    theme: 'gj-dark',
                    language: 'javascript'
                };

                this.editor = monaco.editor.create(this.$refs.editor, {
                    value: JSON.stringify(this.value, '', '    '),
                    ..._options
                });

                this.editor.onDidChangeModelContent(event => {
                    if (!this.__preventTriggerChangeEvent) {
                        // this.$emit('input', JSON.parse(this.getValue()));
                    }
                });

                this.$emit('ready', this.editor);

                this.refresh();

                this.$nextTick(() => {
                    this.editor.onDidBlurEditorText(e => {
                        if (!this.__preventTriggerChangeEvent) {
                            this.$emit('input', JSON.parse(this.getValue()));
                        }
                    });
                });

                this.bus.$on('prop_change', () => {
                    this.getValue() && this.$emit('input', JSON.parse(this.getValue()));
                });
            },
            refresh() {
                this.editor.layout();
                // this.editor.remeasureFonts();
            },
            setValue(value) {
                this.editor.setValue(value);
            },
            getValue() {
                return this.editor.getValue();
            },
            handerCodeChange(value) {
                const model = this.editor.getModel();
                if (this.value && this.value !== this.editor.getValue()) {
                    this.__preventTriggerChangeEvent = true;
                    this.editor.pushUndoStop();
                    model.pushEditOperations(
                        [],
                        [
                            {
                                range: model.getFullModelRange(),
                                text: value
                            }
                        ]
                    );
                    this.editor.pushUndoStop();
                    this.refresh();
                    this.__preventTriggerChangeEvent = false;
                }
            },
            insertContent(text) {
                if (this.editor) {
                    let selection = this.editor.getSelection();
                    let range = new monaco.Range(
                        selection.startLineNumber,
                        selection.startColumn,
                        selection.endLineNumber,
                        selection.endColumn
                    );
                    let id = { major: 1, minor: 1 };
                    let op = { identifier: id, range: range, text: text, forceMoveMarkers: true };
                    this.editor.executeEdits(this.root, [op]);
                    this.editor.focus();
                }
            },
            destroy() {
                this.editor.dispose();
                const model = this.editor.getModel();
                if (model) {
                    model.dispose();
                }
            }
        }
    };
</script>
 
<style lang="scss" scoped>
    .code-editor {
        position: relative;

        ::v-deep > .monaco-editor {
        }
    }
</style>