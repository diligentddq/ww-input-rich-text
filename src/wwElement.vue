<template>
    <div
        class="ww-rich-text"
        :class="{ '-readonly': isReadonly }"
        :style="isEditing && 'pointer-events: none;'"
        data-capture
    >
        <template v-if="richEditor">
            <div class="ww-rich-text__menu" v-if="!hideMenu && !content.customMenu" :style="menuStyles">
                <!-- Text type (normal, ...) -->
                <select id="rich-size" v-model="currentTextType" :disabled="!isEditable" v-if="menu.textType">
                    <option v-for="option in textTypeOptions" :value="option.value">{{ option.label }}</option>
                </select>

                <span class="separator" v-if="menu.textType"></span>

                <!-- Bold, Italic, Underline -->
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="toggleBold"
                    :class="{ 'is-active': richEditor.isActive('bold') }"
                    :disabled="!isEditable"
                    v-if="menu.bold"
                >
                    <i class="fas fa-bold"></i>
                </button>
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="toggleItalic"
                    :class="{ 'is-active': richEditor.isActive('italic') }"
                    :disabled="!isEditable"
                    v-if="menu.italic"
                >
                    <i class="fas fa-italic"></i>
                </button>
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="toggleUnderline"
                    :class="{ 'is-active': richEditor.isActive('underline') }"
                    :disabled="!isEditable"
                    v-if="menu.underline"
                >
                    <i class="fas fa-underline"></i>
                </button>
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="toggleStrike"
                    :class="{ 'is-active': richEditor.isActive('strike') }"
                    :disabled="!isEditable"
                    v-if="menu.strike"
                >
                    <i class="fas fa-strikethrough"></i>
                </button>

                <!-- Show the separator only if at least one of the previous block are visible -->
                <span class="separator" v-if="menu.bold || menu.italic || menu.underline || menu.strike"></span>

                <!-- Text align -->
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="setTextAlign('left')"
                    :class="{ 'is-active': richEditor.isActive({ textAlign: 'left' }) }"
                    :disabled="!isEditable"
                    v-if="menu.alignLeft"
                >
                    <i class="fas fa-align-left"></i>
                </button>

                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="setTextAlign('center')"
                    :class="{ 'is-active': richEditor.isActive({ textAlign: 'center' }) }"
                    :disabled="!isEditable"
                    v-if="menu.alignCenter"
                >
                    <i class="fas fa-align-center"></i>
                </button>

                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="setTextAlign('right')"
                    :class="{ 'is-active': richEditor.isActive({ textAlign: 'right' }) }"
                    :disabled="!isEditable"
                    v-if="menu.alignRight"
                >
                    <i class="fas fa-align-right"></i>
                </button>

                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="setTextAlign('justify')"
                    :class="{ 'is-active': richEditor.isActive({ textAlign: 'justify' }) }"
                    :disabled="!isEditable"
                    v-if="menu.alignJustify"
                >
                    <i class="fas fa-align-justify"></i>
                </button>

                <span
                    class="separator"
                    v-if="menu.alignLeft || menu.alignCenter || menu.alignRight || menu.alignJustify"
                ></span>

                <!-- Color -->
                <label
                    class="ww-rich-text__menu-item"
                    :for="`rich-color-${randomUid}`"
                    @click="richEditor.commands.focus()"
                    v-if="menu.textColor"
                >
                    <i class="fas fa-palette"></i>
                    <input
                        :id="`rich-color-${randomUid}`"
                        type="color"
                        @input="setColor($event.target.value)"
                        :value="richEditor.getAttributes('textStyle').color"
                        style="display: none"
                        :disabled="!isEditable"
                    />
                </label>

                <span class="separator" v-if="menu.textColor"></span>

                <!-- List (Bullet, number) -->
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="toggleBulletList"
                    :class="{ 'is-active': richEditor.isActive('bulletList') }"
                    :disabled="!isEditable"
                    v-if="menu.bulletList"
                >
                    <i class="fas fa-list-ul"></i>
                </button>
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="toggleOrderedList"
                    :class="{ 'is-active': richEditor.isActive('orderedList') }"
                    :disabled="!isEditable"
                    v-if="menu.orderedList"
                >
                    <i class="fas fa-list-ol"></i>
                </button>
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="toggleTaskList"
                    :class="{ 'is-active': richEditor.isActive('taskList') }"
                    :disabled="!isEditable"
                    v-if="menu.taskList"
                >
                    <i class="fas fa-check-square"></i>
                </button>

                <span class="separator" v-if="menu.bulletList || menu.orderedList || menu.taskList"></span>

                <!-- Link -->
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="setLink()"
                    :class="{ 'is-active': richEditor.isActive('link') }"
                    :disabled="!isEditable"
                    v-if="menu.link"
                >
                    <i class="fas fa-link"></i>
                </button>

                <!-- Image -->
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="setImage()"
                    :disabled="!isEditable"
                    v-if="menu.image"
                >
                    <i class="fas fa-image"></i>
                </button>

                <!-- Code -->
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="toggleCodeBlock"
                    :class="{ 'is-active': richEditor.isActive('codeBlock') }"
                    :disabled="!isEditable"
                    v-if="menu.codeBlock"
                >
                    <i class="fas fa-code"></i>
                </button>

                <!-- Quote -->
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="toggleBlockquote"
                    :class="{ 'is-active': richEditor.isActive('blockquote') }"
                    :disabled="!isEditable"
                    v-if="menu.blockquote"
                >
                    <i class="fas fa-quote-left"></i>
                </button>

                <span class="separator" v-if="menu.link || menu.image || menu.codeBlock || menu.blockquote"></span>

                <!-- Table -->
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="insertTable"
                    :disabled="!isEditable"
                    v-if="menu.table"
                >
                    <i class="fas fa-table"></i>
                </button>

                <!-- Undo/Redo -->
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="undo"
                    :disabled="!isEditable"
                    v-if="menu.undo"
                >
                    <i class="fas fa-undo"></i>
                </button>
                <button
                    type="button"
                    class="ww-rich-text__menu-item"
                    @click="redo"
                    :disabled="!isEditable"
                    v-if="menu.redo"
                >
                    <i class="fas fa-redo"></i>
                </button>
            </div>
            <wwElement class="ww-rich-text__menu" v-else-if="content.customMenu" v-bind="content.customMenuElement" />
            <editor-content :editor="richEditor" :style="richStyles" />
        </template>
    </div>
</template>

<script>
import { Editor, EditorContent } from '@tiptap/vue-3';
import StarterKit from '@tiptap/starter-kit';
import Mention from '@tiptap/extension-mention';
import TextStyle from '@tiptap/extension-text-style';
import Color from '@tiptap/extension-color';
import Link from '@tiptap/extension-link';
import Placeholder from '@tiptap/extension-placeholder';
import Underline from '@tiptap/extension-underline';
import Image from '@tiptap/extension-image';
import TaskItem from '@tiptap/extension-task-item';
import TextAlign from '@tiptap/extension-text-align';
import TaskList from '@tiptap/extension-task-list';
import { Table } from '@tiptap/extension-table';
import { TableRow } from '@tiptap/extension-table-row';
import { TableCell } from '@tiptap/extension-table-cell';
import { TableHeader } from '@tiptap/extension-table-header';
import { Markdown } from 'tiptap-markdown';
import { computed } from 'vue';
import { mergeAttributes } from '@tiptap/core';
import suggestion from './suggestion.js';

function extractMentions(acc, currentNode) {
    if (currentNode.type === 'mention') {
        acc.push(currentNode.attrs.id);
        return acc;
    } else if (currentNode.content) {
        return currentNode.content.reduce(extractMentions, acc);
    } else {
        return acc;
    }
}

const TAGS_MAP = {
    p: 0,
    h1: 1,
    h2: 2,
    h3: 3,
    h4: 4,
    h5: 5,
    h6: 6,
};

const CustomTable = Table.extend({
    addAttributes() {
        return {
            style: {
                default:
                    'margin: 0; width: 100%; border-collapse: collapse; box-sizing: border-box; text-indent: initial; border-spacing: 0;',
            },
        };
    },
    renderHTML({ node, HTMLAttributes }) {
        return ['table', mergeAttributes(this.options.HTMLAttributes, HTMLAttributes), 0];
    },
});

const CustomTableCell = TableCell.extend({
    addAttributes() {
        return {
            ...this.parent?.(),
            style: {
                default: 'border: 1px solid #d1cfd7; padding: 8px;',
                parseHTML: element => element.getAttribute('style') || null,
                renderHTML: attributes => {
                    return attributes.style ? { style: attributes.style } : {};
                },
            },
        };
    },
    renderHTML({ node, HTMLAttributes }) {
        return ['td', mergeAttributes(this.options.HTMLAttributes, HTMLAttributes), 0];
    },
});

const CustomTableHeader = TableHeader.extend({
    addAttributes() {
        return {
            ...this.parent?.(),
            style: {
                default: 'border: 1px solid #d1cfd7; padding: 8px; background-color: #f7f7fa;',
                parseHTML: element => element.getAttribute('style') || null,
                renderHTML: attributes => {
                    return attributes.style ? { style: attributes.style } : {};
                },
            },
        };
    },
    renderHTML({ node, HTMLAttributes }) {
        return ['th', mergeAttributes(this.options.HTMLAttributes, HTMLAttributes), 0];
    },
});

export default {
    components: {
        EditorContent,
    },
    props: {
        content: { type: Object, required: true },
        uid: { type: String, required: true },
        wwElementState: { type: Object, required: true },
        /* wwEditor:start */
        wwEditorState: { type: Object, required: true },
        wwFrontState: { type: Object, required: true },
        /* wwEditor:end */
    },
    emits: ['trigger-event', 'update:content:effect', 'update:sidepanel-content'],
    setup(props, { emit }) {
        const { value: variableValue, setValue } = wwLib.wwVariable.useComponentVariable({
            uid: props.uid,
            name: 'value',
            type: 'string',
            defaultValue: computed(() => String(props.content.initialValue || '')),
        });

        const { value: variableMentions, setValue: setMentions } = wwLib.wwVariable.useComponentVariable({
            uid: props.uid,
            name: 'mentions',
            type: 'array',
            defaultValue: [],
            readonly: true,
        });

        const { value: states, setValue: setStates } = wwLib.wwVariable.useComponentVariable({
            uid: props.uid,
            name: 'states',
            type: 'object',
            defaultValue: {},
            readonly: true,
        });

        /* wwEditor:start */
        const { createElement } = wwLib.useCreateElement();
        /* wwEditor:end */

        const randomUid = wwLib.wwUtils.getUid();

        return {
            variableValue,
            setValue,
            variableMentions,
            setMentions,
            states,
            setStates,
            randomUid,
            /* wwEditor:start */
            createElement,
            /* wwEditor:end */
        };
    },
    data: () => ({
        richEditor: null,
        loading: false,
    }),
        methods: {
        loadEditor() {
            if (this.loading) return;
            this.loading = true;
            if (this.richEditor) this.richEditor.destroy();

            this.richEditor = new Editor({
                content: String(this.content.initialValue || ''),
                editable: this.isEditable,
                autofocus: this.editorConfig.autofocus,
                onFocus: ({ editor, event }) => {
                    this.$emit('trigger-event', { name: 'focus', event: { editor, event } });
                },
                onBlur: ({ editor, event }) => {
                    this.$emit('trigger-event', { name: 'blur', event: { editor, event } });
                },
                extensions: [
                    StarterKit,
                    Link.configure({
                        HTMLAttributes: {
                            rel: 'noopener noreferrer',
                        },
                    }),
                    TextStyle,
                    Color,
                    Underline,
                    TaskList,
                    TaskItem.configure({
                        nested: true,
                    }),
                    TextAlign.configure({
                        types: ['heading', 'paragraph'],
                    }),
                    Placeholder.configure({
                        placeholder: this.editorConfig.placeholder,
                    }),
                    Markdown.configure({ breaks: true }),
                    Image.configure({ ...this.editorConfig.image }),
                    // Conditionally add Mention extension
                    ...(this.editorConfig.mention.enabled
                        ? [
                              Mention.configure({
                                  HTMLAttributes: {
                                      class: 'mention',
                                  },
                                  suggestion: {
                                      items: ({ query }) =>
                                          this.editorConfig.mention.list
                                              .filter(({ label }) =>
                                                  label.toLowerCase().startsWith(query.toLowerCase()),
                                              )
                                              .slice(0, this.mentionListLength),
                                      render: suggestion.render,
                                      allowSpaces: this.editorConfig.mention.allowSpaces,
                                      char: this.editorConfig.mention.char,
                                  },
                              }),
                          ]
                        : []),
                    CustomTable.configure({
                        resizable: true,
                        HTMLAttributes: {
                            style:
                                'margin: 0; width: 100%; border-collapse: collapse; box-sizing: border-box; text-indent: initial; border-spacing: 0;',
                        },
                    }),
                    TableRow,
                    CustomTableHeader,
                    CustomTableCell,
                ],
                onCreate: () => {
                    this.setValue(this.getContent());
                    this.setMentions(this.richEditor.getJSON().content.reduce(extractMentions, []));
                },
                onUpdate: this.handleOnUpdate,
                editorProps: {
                    handleClickOn: (view, pos, node) => {
                        if (node.type.name === 'mention') {
                            this.$emit('trigger-event', {
                                name: 'mention:click',
                                event: { mention: { id: node.attrs.id, label: node.attrs.label } },
                            });
                        }
                    },
                },
            });
            this.loading = false;
        },
        handleOnUpdate() {
            const htmlValue = this.getContent();
            if (this.variableValue === htmlValue) return;
            this.setValue(htmlValue);
            if (this.content.debounce) {
                this.isDebouncing = true;
                if (this.debounce) {
                    clearTimeout(this.debounce);
                }
                this.debounce = setTimeout(() => {
                    this.$emit('trigger-event', { name: 'change', event: { value: this.variableValue } });
                    this.isDebouncing = false;
                }, this.delay);
            } else {
                this.$emit('trigger-event', { name: 'change', event: { value: this.variableValue } });
            }
            this.setMentions(this.richEditor.getJSON().content.reduce(extractMentions, []));
        },
        setLink(url) {
            if (this.richEditor.isActive('link')) {
                this.richEditor.chain().focus().unsetLink().run();
                return;
            }

            const previousUrl = this.richEditor.getAttributes('link').href;
            const selectedUrl = url ?? window.prompt('URL', previousUrl);

            // cancelled
            if (selectedUrl === null) {
                return;
            }

            // empty
            if (selectedUrl === '') {
                this.richEditor.chain().focus().extendMarkRange('link').unsetLink().run();
                return;
            }

            // update link
            this.richEditor.chain().focus().extendMarkRange('link').setLink({ href: selectedUrl }).run();
        },
        setImage(src, alt = '', title = '') {
            if (this.content.customMenu) {
                this.richEditor.commands.setImage({ src, alt, title });
            } else {
                let url;
                /* wwEditor:start */
                url = wwLib.getEditorWindow().prompt('Image URL');
                /* wwEditor:end */
                /* wwFront:start */
                url = wwLib.getFrontWindow().prompt('Image URL');
                /* wwFront:end */

                if (!url) return;
                this.richEditor.chain().focus().setImage({ src: url }).run();
            }
        },
        focusEditor() {
            this.richEditor.chain().focus().run();
        },
        setTag(tag) {
            if (typeof tag === 'string') {
                tag = tag.toLocaleLowerCase().trim();
                if (tag in TAGS_MAP) tag = TAGS_MAP[tag];
            }
            if (tag === 0) {
                this.richEditor.chain().focus().setParagraph().run();
            } else {
                this.richEditor.chain().focus().toggleHeading({ level: Number(tag) }).run();
            }
        },
        toggleUnderline() {
            this.richEditor.chain().focus().toggleUnderline().run();
        },
        toggleBold() {
            this.richEditor.chain().focus().toggleBold().run();
        },
        toggleItalic() {
            this.richEditor.chain().focus().toggleItalic().run();
        },
        toggleStrike() {
            this.richEditor.chain().focus().toggleStrike().run();
        },
        setTextAlign(textAlign) {
            this.richEditor.chain().focus().setTextAlign(textAlign).run();
        },
        setColor(color) {
            this.richEditor.chain().focus().setColor(color).run();
        },
        toggleBulletList() {
            this.richEditor.chain().focus().toggleBulletList().run();
        },
        toggleOrderedList() {
            this.richEditor.chain().focus().toggleOrderedList().run();
        },
        toggleTaskList() {
            this.richEditor.chain().focus().toggleTaskList().run();
        },
        toggleCodeBlock() {
            this.richEditor.chain().focus().toggleCodeBlock().run();
        },
        toggleBlockquote() {
            this.richEditor.chain().focus().toggleBlockquote().run();
        },
        undo() {
            this.richEditor.chain().undo().run();
        },
        redo() {
            this.richEditor.chain().redo().run();
        },
        getContent() {
            // Return HTML to preserve inline styles
            return this.richEditor.getHTML();
        },
        insertTable() {
            if (!this.richEditor.can().insertTable()) {
                alert('Cannot insert table here!');
                return;
            }
            this.richEditor.chain().focus().insertTable({ rows: 3, cols: 3, withHeaderRow: true }).run();
        },
    },
    mounted() {
        this.loadEditor();
    },
    beforeUnmount() {
        if (this.richEditor) this.richEditor.destroy();
    },
};
</script>

<style lang="scss">
/* Your existing styles remain unchanged */
</style>
