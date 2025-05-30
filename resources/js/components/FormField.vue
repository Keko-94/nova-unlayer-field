<template>
    <Readonly v-if="isReadonly"
                 :field="field"
    />
    <DefaultField v-else
        :field="currentField"
        :errors="errors"
        :show-help-text="showHelpText"
        :full-width-content="true"
    >
        <template #field>
            <div class="fullscreenable">
                <div class="unlayerControls flex">
                    <button
                            id="fullscreenToggleButton"
                            class="text-xs bg-90 font-semibold rounded-sm px-4 py-1 m-1 border"
                            @click="toggleFullscreen"
                            type="button">
                        {{ __('Enter fullscreen') }}
                    </button>
                </div>

                <UnlayerEditor
                    class="form-input-bordered"
                    :style="{minHeight: containerHeight}"
                    ref="editor"
                    @load="editorLoaded"
                    :locale=currentField.config.locale
                    :projectId=currentField.config.projectId
                    :templateId="currentField.value ? null : currentField.config.templateId"
                    :options=unProxy(currentField.config)
                    :key=currentField.config.key
                />
            </div>
        </template>
    </DefaultField>
</template>

<script>
    import UnlayerEditor from './UnlayerEditor.vue';
    import Readonly from './Readonly.vue';
    import { DependentFormField, HandlesValidationErrors } from 'laravel-nova'

    const defaultHeight = '700px';

    export default {
        mixins: [DependentFormField, HandlesValidationErrors],

        components: {
            UnlayerEditor,
            Readonly
        },

        props: ['resourceName', 'resourceId', 'field'],

        computed: {
            containerHeight: function () {
                return this.currentField.height || defaultHeight;
            },
            isReadonly: function() {
                return
                    this.currentField.readonly ||
                    (this.currentField.hasOwnProperty('extraAttributes') && this.currentField.extraAttributes.hasOwnProperty('readonly') && this.currentField.extraAttributes.readonly) ||
                    false
            }
        },

        methods: {
            /** Register listeners, load initial template, etc. */
            editorLoaded() {
                if (this.currentField.value !== null) {
                    this.$refs.editor.loadDesign(this.unProxy(this.currentField.value));
                }

                /** @see https://docs.unlayer.com/docs/events */
                window.unlayer.addEventListener('design:loaded', this.handleDesignLoaded);
                window.unlayer.addEventListener('design:updated', this.handleDesignUpdated);
                window.unlayer.addEventListener('image:uploaded', this.handleImageUploaded);

                this.loadPlugins(this.currentField.plugins);
            },

            /**
             * @param {Proxy<Object>} proxyInstance
             * @return {Object}
             */
            unProxy(proxyInstance) {
                return JSON.parse(JSON.stringify(proxyInstance));
            },

            /**
             * @param {Array<string>} pluginsUrls
             */
            loadPlugins(pluginsUrls) {
                if (window.unlayer.plugins === undefined) {
                    window.unlayer.plugins = [];
                }

                pluginsUrls.forEach(pluginUrl => {
                    const script = document.createElement('script');
                    script.setAttribute('src', pluginUrl);
                    document.head.appendChild(script);
                });
            },

            /**
             * Fill the given FormData object with the field's internal value.
             * Nova runs it before submission.
             * @property {FormData} formData
             */
            fill(formData) {
                formData.append(this.currentField.attribute, JSON.stringify(this.design));
                formData.append(`${this.currentField.attribute}_html`, this.html);
            },

            /**
             * @param {{design: Object}} loadedDesign
             */
            handleDesignLoaded(loadedDesign) {
                this.$refs.editor.exportHtml((editorData) => {
                    this.design = editorData.design;
                    this.html = editorData.html;
                });

                Nova.$emit('unlayer:design:loaded', {
                    inputName: this.currentField.attribute,
                    payload: loadedDesign,
                });
            },

            /**
             * Generate a design where we use replace a changed node
             * (usually updated by plugins) by it's new state
             * @param {Object} updatedNode
             * @param {Object} design
             * @returns {Object}
             */
            getDesignWithUpdatedNode(updatedNode, design) {
                const htmlIdOfChangedNode = updatedNode.values._meta.htmlID;

                design.body.rows.find((row, rowIndex) => {
                    return row.columns.find((column, columnIndex) => {
                        return column.contents.find((currentNode, contentIndex) => {
                            if (currentNode.values._meta.htmlID !== htmlIdOfChangedNode) {
                                return false;
                            }

                            design.body.rows[rowIndex].columns[columnIndex].contents[contentIndex] = updatedNode;
                            return true;
                        }) === true;
                    }) === true;
                });

                return design;
            },

            /**
             * @param {{item: Object, type: string, changes: ?Object}} changeLog
             */
            handleDesignUpdated(changeLog) {
                const originalChangedItemAsString = JSON.stringify(changeLog.item);

                /** @type {Object} */
                const updatedByPluginsNode = Object.values(window.unlayer.plugins).reduce((prev, pluginFn) => {
                    return pluginFn(prev, changeLog.type, changeLog.changes);
                }, changeLog.item);

                const updatedChangeLogAsString = JSON.stringify(updatedByPluginsNode);
                if (originalChangedItemAsString === updatedChangeLogAsString) {
                    this.$refs.editor.exportHtml((editorData) => {
                        this.design = editorData.design;
                        this.html = editorData.html;
                    });
                } else {
                    /**
                     * 1. Get current design [using exportHtml()]
                     * 2. Load updated (by plugins) design [using loadDesign()]
                     * 3. Store export HTML [need to use another exportHtml() to get final HTML]
                     */
                    this.$refs.editor.exportHtml((editorData) => {
                        const design = this.getDesignWithUpdatedNode(updatedByPluginsNode, editorData.design);
                        this.$refs.editor.loadDesign(design);

                        this.$refs.editor.exportHtml((editorData) => {
                            this.design = editorData.design;
                            this.html = editorData.html;
                        });
                    });
                }

                Nova.$emit('unlayer:design:updated', {
                    inputName: this.currentField.attribute,
                    payload: changeLog,
                });
            },

            /**
             * @param {{image:string, url:string, width:int, height:int}} imageData
             */
            handleImageUploaded(imageData) {
                Nova.$emit('unlayer:image:uploaded', {
                    inputName: this.currentField.attribute,
                    payload: imageData,
                });
            },

            toggleFullscreen() {
                // toggle scrolling of the page
                document.body.classList.toggle('overflow-hidden');

                const unlayerEditorContainer = this.$el.querySelector(`.fullscreenable`);
                unlayerEditorContainer.classList.toggle('z-50'); // increase z-index
                unlayerEditorContainer.classList.toggle('fullscreen');

                const controls = this.$el.querySelector('.unlayerControls');
                controls.classList.toggle('stickyControls');

                const trans = function (key) {
                    return Nova.config('translations')[key];
                };

                const toggleButton = controls.querySelector(`#fullscreenToggleButton`);
                unlayerEditorContainer.classList.contains('fullscreen')
                    ? toggleButton.innerText = trans('Exit fullscreen')
                    : toggleButton.innerText = trans('Enter fullscreen');
            },

        },
    }
</script>

<style scoped>
    .fullscreen {
        position: fixed;
        left: 1vw;
        top: 1vh;
        width: 98vw !important;
        height: 98vh !important;
    }

    .stickyControls {
        position: fixed;
        left: 1vw;
        top: 1vh;
        z-index: 51;
    }
</style>
