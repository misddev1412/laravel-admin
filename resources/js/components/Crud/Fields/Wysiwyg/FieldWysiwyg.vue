<template>
    <lit-base-field
        :field="field"
        :model="model"
        v-slot:default="{ state }"
        v-on="$listeners"
    >
        <div class="lit-field-wysiwyg__css" v-if="field.css">
            <v-style v-html="prependCssSelectors(field.css)"></v-style>
        </div>
        
        <template v-if="!field.readonly">
            <div
                
                :class="state === false ? 'form-control is-invalid' : ''"
                v-if="editor"
            >
              
            
                <ckeditor :editor="editorContent" 
                :id="identifier" 
                v-if="!editRaw" 
                v-model="content" 
                :config="editorConfig"></ckeditor>


                <b-form-textarea
                    class="lit-field-wysiwyg_raw"
                    v-model="valueCopy"
                    v-if="editRaw"
                    rows="3"
                ></b-form-textarea>
            </div>
        </template>
        <template v-else>
            <div v-html="value"></div>
        </template>

        <slot />
    </lit-base-field>
</template>

<script>
import Vue from 'vue';

import CKEditor from '@ckeditor/ckeditor5-vue2';
// import CKFinder from '@ckeditor/ckeditor5-ckfinder/src/ckfinder';

Vue.use( CKEditor );
// import ClassicEditor from '@ckeditor/ckeditor5-build-classic';
import ClassicEditor from '@ckeditor/ckeditor5-editor-classic/src/classiceditor'
import { Editor, EditorContent, EditorMenuBar } from 'tiptap';
import {
    Blockquote,
    CodeBlock,
    HardBreak,
    Heading,
    HorizontalRule,
    OrderedList,
    BulletList,
    ListItem,
    TodoItem,
    TodoList,
    Bold,
    Code,
    Italic,
    Strike,
    Underline,
    History,
    Table,
    TableHeader,
    TableCell,
    TableRow,
} from 'tiptap-extensions';
import CustomLink from './Nodes/CustomLink';
import FontColor from './Nodes/FontColor';
import { mapGetters } from 'vuex';

export default {
    name: 'FieldWysiwyg',
    components: {
        EditorContent,
        CKEditor,
        EditorMenuBar,
        'v-style': {
            render: function(createElement) {
                return createElement('style', this.$slots.default);
            },
        },
    },
    props: {
        field: {
            type: Object,
            required: true,
        },
        model: {
            required: true,
            type: Object,
        },
        value: {
            required: true,
        },
    },
    data() {
        return {
            content: null,
            editor: null,
            editRaw: false,
            linkUrl: null,
            target: null,
            valueCopy: _.clone(this.value),
            editorUrl: "https://cdn.ckeditor.com/4.14.1/full-all/ckeditor.js",
            editorData: '',
            editorContent: ClassicEditor,
            editorConfig: {
           

            }

        };
    },
    beforeMount() {
        this.init();
    },
    mounted() {
        this.editor.setContent(this.value);
        this.content = this.value
        this.editor.on('update', ({ getHTML }) => {
            this.$emit('input', getHTML());
            this.valueCopy = _.clone(getHTML());
        });


        Lit.bus.$on('languageChanged', () => {
            this.$nextTick(() => {
                this.valueCopy = _.clone(this.value);
                this.content = this.value
                // this.editor.setContent(this.value);
            });
        });
    },
    beforeDestroy() {
        this.editor.destroy();
    },
    watch: {
        valueCopy(val) {
            if (this.editRaw) {
                let data = this.stripScripts(val);
                this.editor.setContent(data);
                this.$emit('input', data);
            }
        },
        content(val) {
            console.log(val)
            this.valueCopy = _.clone(val);

            this.$emit('input', val);

        },
    },
    methods: {
        init() {
            this.editor = new Editor({
                disablePasteRules: !this.field.enablePasteRules,
                disableInputRules: !this.field.enableInputRules,
                extensions: [
                    new Blockquote(),
                    new CodeBlock(),
                    new HardBreak(),
                    new Heading({ levels: this.headingLevels }),
                    new HorizontalRule(),
                    new BulletList(),
                    new OrderedList(),
                    new ListItem(),
                    new TodoItem(),
                    new TodoList(),
                    new Bold(),
                    new Code(),
                    new Italic(),
                    new CustomLink(),
                    new Strike(),
                    new Underline(),
                    new History(),
                    new FontColor(),
                    new Table({ resizable: true }),
                    new TableHeader(),
                    new TableCell(),
                    new TableRow(),
                ],
            });
        },
        // credits:
        // https://stackoverflow.com/questions/6659351/removing-all-script-tags-from-html-with-js-regular-expression/6660151
        stripScripts(s) {
            var div = document.createElement('div');
            div.innerHTML = s;
            var scripts = div.getElementsByTagName('script');
            var i = scripts.length;
            while (i--) {
                scripts[i].parentNode.removeChild(scripts[i]);
            }
            return div.innerHTML;
        },
        format(isActive) {
            if (isActive.paragraph()) {
                return 'Paragraph';
            }

            for (let index = 0; index < this.headingLevels.length; index++) {
                const el = this.headingLevels[index];
                if (isActive.heading({ level: el })) {
                    return `H${el}`;
                }
            }
        },
        hasControl(control) {
            if (this.field.only) {
                return _.includes(this.field.only, control);
            }
            return _.includes(this.lit_config.fields.wysiwyg.controls, control);
        },
        showLinkMenu(attrs) {
            this.linkUrl = attrs.href;
            this.target = attrs.target;
        },
        setLinkUrl(command) {
            command({ href: this.linkUrl, target: this.target });
            this.linkUrl = null;
            this.target = null;
        },
        setFontColor(command, color) {
            command({ style: `color: ${color}` });
        },
        prependCssSelectors(css) {
            // Prepend every css-selector with the identifier of the editor
            // credit: Ryan Worth
            // https://stackoverflow.com/questions/11161198/prepend-all-css-selectors
            return css.replace(
                /(^(?:\s|[^@{])*?|[},]\s*)(\/\/.*\s+|.*\/\*[^*]*\*\/\s*|@media.*{\s*|@font-face.*{\s*)*([.#]?-?[_a-zA-Z]+[_a-zA-Z0-9-]*)(?=[^}]*{)/g,
                `$1$2 #${this.identifier} $3`
            );
        },
        toggleHtmlView() {
            this.editRaw = !this.editRaw;
        },
    },
    computed: {
        ...mapGetters(['lit_config']),
        identifier() {
            return `${this.field.local_key}-${this.field.route_prefix.replace(
                /\//g,
                '-'
            )}`;
        },
        headingLevels() {
            let levels =
                this.lit_config?.fields?.wysiwyg?.headingLevels || null;

            return levels || [2, 3, 4];
        },
        colors() {
            return (
                this.field.colors ||
                this.lit_config?.fields?.wysiwyg?.colors ||
                null
            );
        },
    },
};
</script>
<style lang="scss">
@import '@lit-sass/_variables';
.lit-field-wysiwyg {
    width: 100%;
    border-radius: $border-radius;
    border: 1px solid $border-color;
    background: white;
    min-height: $input-height;

    padding: $input-padding-x;
    padding-bottom: 0;
    &__menu {
        display: grid;
        grid-template-columns: repeat(auto-fill, 1.75rem);
        grid-auto-rows: 1fr;
        grid-gap: 0.5rem;
        &-dropdown {
            grid-column: 1 / span 5 !important;
        }
    }
    &__content {
        padding-top: $input-padding-x;
        .ProseMirror {
            padding-bottom: 1px;
            &:focus {
                outline: none;
            }
        }
        p {
            line-height: 1.25rem;
            font-size: $input-font-size;
        }
        table,
        th,
        td {
            border: 1px solid $border-color;
        }
        table {
            border-radius: 15px;
            margin-bottom: 1rem;
        }
        th,
        td {
            min-width: 2rem;
            padding: 0.5rem;
            p {
                margin-bottom: 0;
            }
        }
    }

    .dropdown-sm-square.show {
        .dropdown-toggle {
            background: $secondary;
            color: white;
        }
    }
    .lit-color-palette {
        .dropdown-menu {
            width: 14rem;
        }
        &-wrapper {
            display: grid;
            grid-template-columns: repeat(auto-fill, 1.75rem);
            grid-auto-rows: 1fr;
            grid-gap: 0.5rem;
            padding: 0 0.5rem;
        }
    }
    &_raw {
        border: none;
        outline: none !important;
        box-shadow: none !important;
    }
}
</style>
