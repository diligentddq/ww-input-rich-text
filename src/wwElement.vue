<template>
    <!-- Existing template structure -->
    <div class="ww-rich-text" :class="{ '-readonly': isReadonly }" :style="isEditing && 'pointer-events: none;'" data-capture>
        <template v-if="richEditor">
            <div class="ww-rich-text__menu" v-if="!hideMenu && !content.customMenu" :style="menuStyles">
                <!-- Existing buttons... -->
                <!-- Table related buttons -->
                <button type="button" class="ww-rich-text__menu-item" @click="addTable" :disabled="!isEditable">
                    <i class="fas fa-table"></i>
                </button>

              
                <button type="button" class="ww-rich-text__menu-item" @click="deleteTable" :disabled="!isEditable">
                    <i class="fas fa-ban"></i>
                </button>
                <span class="separator" v-if="menu.table"></span>
                <!-- Other buttons for table operations can be added similarly -->
            </div>
            <wwElement class="ww-rich-text__menu" v-else-if="content.customMenu" v-bind="content.customMenuElement" />
            <editor-content :editor="richEditor" :style="richStyles" />
        </template>
    </div>a
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
    import { Markdown } from 'tiptap-markdown';
    import { Table, TableRow, TableCell, TableHeader } from '@tiptap/extension-table';  // New imports for table
    import { computed } from 'vue';
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

            const randomUid = wwLib.wwUtils.getUid();

            return {
                variableValue,
                setValue,
                variableMentions,
                setMentions,
                states,
                setStates,
                randomUid,
            };
        },
        data: () => ({
            richEditor: null,
            loading: false,
        }),

        watch: {
            'content.initialValue'(value) {
                if (value !== this.getContent()) {
                    this.richEditor.commands.setContent(value);
                    this.setValue(value);
                }
                this.$emit('trigger-event', { name: 'initValueChange', event: { value } });

                if (this.isReadonly) this.handleOnUpdate();
            },
            isEditable(value) {
                this.richEditor.setEditable(value);
            },
            variableValue(value, oldValue) {
                if (value !== this.getContent()) this.richEditor.commands.setContent(value);
                if (value !== this.getContent()) this.setValue(this.getContent());
            },
            editorConfig() {
                this.loadEditor();
            },
        },
        computed: {
            isEditing() {
                /* wwEditor:start */
                return this.wwEditorState.editMode === wwLib.wwEditorHelper.EDIT_MODES.EDITION;
                /* wwEditor:end */
                return false;
            },
            editorStates() {
                if (!this.richEditor) return {};
                return {
                    textType: Object.keys(TAGS_MAP).find(key => TAGS_MAP[key] === this.currentTextType),
                    textColor: this.currentColor,
                    bold: this.richEditor.isActive('bold'),
                    italic: this.richEditor.isActive('italic'),
                    underline: this.richEditor.isActive('underline'),
                    strike: this.richEditor.isActive('strike'),
                    bulletList: this.richEditor.isActive('bulletList'),
                    orderedList: this.richEditor.isActive('orderedList'),
                    checkList: this.richEditor.isActive('taskList'),
                    link: this.richEditor.isActive('link'),
                    codeBlock: this.richEditor.isActive('codeBlock'),
                    blockquote: this.richEditor.isActive('blockquote'),
                    textAlign: this.richEditor.isActive({ textAlign: 'left' })
                        ? 'left'
                        : this.richEditor.isActive({ textAlign: 'center' })
                        ? 'center'
                        : this.richEditor.isActive({ textAlign: 'right' })
                        ? 'right'
                        : this.richEditor.isActive({ textAlign: 'justify' })
                        ? 'justify'
                        : false,
                };
            },
            currentColor() {
                if (this.richEditor.getAttributes('textStyle')?.color)
                    return this.richEditor.getAttributes('textStyle')?.color;
                else if (this.richEditor.isActive('link')) return this.content.a.color;
                else if (this.richEditor.isActive('codeBlock')) return this.content.code.color;
                else if (this.richEditor.isActive('blockquote')) return this.content.blockquote.color;
                else return this.content[Object.keys(TAGS_MAP).find(key => TAGS_MAP[key] === this.currentTextType)]?.color;
            },
            mentionList() {
                const data = wwLib.wwCollection.getCollectionData(this.content.mentionList);
                if (!Array.isArray(data)) return [];
                return data.map(mention => ({
                    id: wwLib.resolveObjectPropertyPath(mention, this.content.mentionIdPath || 'id') || '',
                    label: wwLib.resolveObjectPropertyPath(mention, this.content.mentionLabelPath || 'label') || '',
                }));
            },
            mentionListLength() {
                if (!this.content.mentionListLength || isNaN(this.content.mentionListLength)) return 5;
                return this.content.mentionListLength;
            },
            isReadonly() {
                return this.wwElementState.props.readonly === undefined
                    ? this.content.readonly
                    : this.wwElementState.props.readonly;
            },
            isEditable() {
                return !this.isReadonly && this.content.editable;
            },
            hideMenu() {
                return this.content.hideMenu || this.isReadonly;
            },
            menu() {
                return {
                    textType: this.content.parameterTextType ?? true,
                    bold: this.content.parameterBold ?? true,
                    italic: this.content.parameterItalic ?? true,
                    underline: this.content.parameterUnderline ?? true,
                    strike: this.content.parameterStrike ?? true,
                    alignLeft: this.content.parameterAlignLeft ?? false,
                    alignCenter: this.content.parameterAlignCenter ?? false,
                    alignRight: this.content.parameterAlignRight ?? false,
                    alignJustify: this.content.parameterAlignJustify ?? false,
                    textColor: this.content.parameterTextColor ?? true,
                    bulletList: this.content.parameterBulletList ?? true,
                    orderedList: this.content.parameterOrderedList ?? true,
                    taskList: this.content.parameterTaskList ?? false,
                    link: this.content.parameterLink ?? true,
                    image: this.content.parameterImage ?? false,
                    codeBlock: this.content.parameterCodeBlock ?? true,
                    blockquote: this.content.parameterQuote ?? true,
                    undo: this.content.parameterUndo ?? true,
                    redo: this.content.parameterRedo ?? true,
                    table: this.content.parameterTable ?? true,  // Add table parameter
                };
            },
            editorConfig() {
                return {
                    placeholder: this.content.placeholder,
                    autofocus: this.content.autofocus,
                    image: {
                        inline: this.content.img?.inline,
                        allowBase64: true,
                    },
                    mention: {
                        enabled: this.content.enableMention,
                        list: this.mentionList,
                        allowSpaces: this.content.mentionAllowSpaces,
                        char: this.content.mentionChar,
                    },
                };
            },
            currentTextType: {
                get() {
                    const currentType = this.textTypeOptions.find(option => option.active);
                    return currentType ? currentType.value : 0;
                },
                set(value) {
                    this.setTag(value);
                },
            },
            textTypeOptions() {
                if (!this.richEditor) return [];
                return [
                    { label: 'Paragraph', value: 0, active: this.richEditor.isActive('paragraph') },
                    { label: 'Heading 1', value: 1, active: this.richEditor.isActive('heading', { level: 1 }) },
                    { label: 'Heading 2', value: 2, active: this.richEditor.isActive('heading', { level: 2 }) },
                    { label: 'Heading 3', value: 3, active: this.richEditor.isActive('heading', { level: 3 }) },
                    { label: 'Heading 4', value: 4, active: this.richEditor.isActive('heading', { level: 4 }) },
                    { label: 'Heading 5', value: 5, active: this.richEditor.isActive('heading', { level: 5 }) },
                    { label: 'Heading 6', value: 6, active: this.richEditor.isActive('heading', { level: 6 }) },
                ];
            },
            menuStyles() {
                return {
                    '--menu-color': this.content.menuColor,
                };
            },
            richStyles() {
                return {
                    overflow: 'auto',
                    // H1
                    '--h1-fontSize': this.content.h1.fontSize,
                    '--h1-fontFamily': this.content.h1.fontFamily,
                    '--h1-fontWeight': this.content.h1.fontWeight,
                    '--h1-textAlign': this.content.h1.textAlign,
                    '--h1-color': this.content.h1.color,
                    '--h1-lineHeight': this.content.h1.lineHeight,
                    '--h1-margin-top': this.content.h1.marginTop,
                    '--h1-margin-bottom': this.content.h1.marginBottom,
                    // H2
                    '--h2-fontSize': this.content.h2.fontSize,
                    '--h2-fontFamily': this.content.h2.fontFamily,
                    '--h2-fontWeight': this.content.h2.fontWeight,
                    '--h2-textAlign': this.content.h2.textAlign,
                    '--h2-color': this.content.h2.color,
                    '--h2-lineHeight': this.content.h2.lineHeight,
                    '--h2-margin-top': this.content.h2.marginTop,
                    '--h2-margin-bottom': this.content.h2.marginBottom,
                    // H3
                    '--h3-fontSize': this.content.h3.fontSize,
                    '--h3-fontFamily': this.content.h3.fontFamily,
                    '--h3-fontWeight': this.content.h3.fontWeight,
                    '--h3-textAlign': this.content.h3.textAlign,
                    '--h3-color': this.content.h3.color,
                    '--h3-lineHeight': this.content.h3.lineHeight,
                    '--h3-margin-top': this.content.h3.marginTop,
                    '--h3-margin-bottom': this.content.h3.marginBottom,
                    // H4
                    '--h4-fontSize': this.content.h4.fontSize,
                    '--h4-fontFamily': this.content.h4.fontFamily,
                    '--h4-fontWeight': this.content.h4.fontWeight,
                    '--h4-textAlign': this.content.h4.textAlign,
                    '--h4-color': this.content.h4.color,
                    '--h4-lineHeight': this.content.h4.lineHeight,
                    '--h4-margin-top': this.content.h4.marginTop,
                    '--h4-margin-bottom': this.content.h4.marginBottom,
                    // H5
                    '--h5-fontSize': this.content.h5.fontSize,
                    '--h5-fontFamily': this.content.h5.fontFamily,
                    '--h5-fontWeight': this.content.h5.fontWeight,
                    '--h5-textAlign': this.content.h5.textAlign,
                    '--h5-color': this.content.h5.color,
                    '--h5-lineHeight': this.content.h5.lineHeight,
                    '--h5-margin-top': this.content.h5.marginTop,
                    '--h5-margin-bottom': this.content.h5.marginBottom,
                    // H6
                    '--h6-fontSize': this.content.h6.fontSize,
                    '--h6-fontFamily': this.content.h6.fontFamily,
                    '--h6-fontWeight': this.content.h6.fontWeight,
                    '--h6-textAlign': this.content.h6.textAlign,
                    '--h6-color': this.content.h6.color,
                    '--h6-lineHeight': this.content.h6.lineHeight,
                    '--h6-margin-top': this.content.h6.marginTop,
                    '--h6-margin-bottom': this.content.h6.marginBottom,
                    // p
                    '--p-fontSize': this.content.p.fontSize,
                    '--p-fontFamily': this.content.p.fontFamily,
                    '--p-fontWeight': this.content.p.fontWeight,
                    '--p-textAlign': this.content.p.textAlign,
                    '--p-color': this.content.p.color,
                    '--p-lineHeight': this.content.p.lineHeight,
                    '--p-margin-top': this.content.p.marginTop,
                    '--p-margin-bottom': this.content.p.marginBottom,
                    // mention
                    '--mention-fontSize': this.content.mention.fontSize,
                    '--mention-fontFamily': this.content.mention.fontFamily,
                    '--mention-fontWeight': this.content.mention.fontWeight,
                    '--mention-color': this.content.mention.color,
                    '--mention-borderSize': this.content.mention.borderSize,
                    '--mention-border-radius': this.content.mention.borderRadius,
                    // a
                    '--a-fontSize': this.content.a.fontSize,
                    '--a-fontFamily': this.content.a.fontFamily,
                    '--a-fontWeight': this.content.a.fontWeight,
                    '--a-textAlign': this.content.a.textAlign,
                    '--a-color': this.content.a.color,
                    '--a-lineHeight': this.content.a.lineHeight,
                    '--a-underline': this.content.a.isUnderline ? 'underline' : 'none',
                    // blockquote
                    '--blockquote-color': this.content.blockquote.color,
                    '--blockquote-border-color': this.content.blockquote.borderColor,
                    '--blockquote-margin-top': this.content.blockquote.marginTop,
                    '--blockquote-margin-bottom': this.content.blockquote.marginBottom,
                    // code
                    '--code-color': this.content.code.color,
                    '--code-bg-color': this.content.code.bgColor,
