<template>
  <div class="mentionable-editor__wrapper" :id="id">
    <div class="editor">
      <editor-content class="editor__content" :editor="editor" />
      <div class="suggestion-list" v-show="showSuggestions" ref="suggestions">
        <template v-if="hasResults">
            <div
              v-for="(item, index) in filteredItems"
              :key="item.id"
              class="suggestion-list__item"
              :class="{ 'is-selected': navigatedItemIndex === index }"
              @click="selectItem(item)"
            >
              <slot name="mention-item" :item="item">
                {{ item.name }}
              </slot>
            </div>
        </template>
        <div v-else class="suggestion-list__item is-empty">
          <slot name="no-results-item">
            No items found
          </slot>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
// yarn add tiptap tiptap-extensions tippy.js
// This component has built-in filter for mentionItems already
import tippy, { sticky } from 'tippy.js'
import { Editor, EditorContent } from 'tiptap'
import {
  HardBreak,
  Mention,
} from 'tiptap-extensions'
export default {
  components: {
    EditorContent,
  },
  props: {
    value: String,
    id: {
      type: String,
      required: true
    },
    mentionValue: {
      type: String,
      default: 'id'
    },
    mentionLabel: {
      type: String,
      default: 'name'
    },
    mentionPrefix: {
      type: String,
      default: '@'
    },
    mentionPostfix: {
      type: String,
      default: ''
    },
    mentionItems: {
      type: Array,
      default() {
        return [
          { id: 1, name: 'Item 1' },
          { id: 2, name: 'Item 2' },
          { id: 3, name: 'Item 3' },
        ]
      }
    }
  },
  data() {
    return {
      editor: null,
      query: null,
      suggestionRange: null,
      filteredItems: [],
      navigatedItemIndex: 0,
      insertMention: () => {},
    }
  },
  async mounted() {
    this.editor = await new Editor({
      onInit: () => {
        // editor is initialized
        // Handle paste from clipboard (both text and image paste)
        document.querySelector('#' + this.id).addEventListener('paste', (event) => {
          // consider the first item (can be easily extended for multiple items)
          if(event.clipboardData.getData('Text')) {
            this.$emit('paste', {
              type: 'text',
              value: event.clipboardData.getData('Text')
            })
          } else {
            var item = event.clipboardData.items[1] ? event.clipboardData.items[1] : event.clipboardData.items[0]
            if (item.type.indexOf("image") === 0) {
              var blob = item.getAsFile()
              var reader = new FileReader()
              reader.onload = (event) => {
                this.$emit('paste', {
                  type: 'image',
                  value: event.target.result
                })
              };
              reader.readAsDataURL(blob)
            }
          }
        });
      },
      // Editor options
      onUpdate: ({ getHTML }) => {
        let div = document.createElement("div")
        div.innerHTML = getHTML()
        this.$emit('input', getHTML())
        this.$emit('change', {
          html: getHTML(),
          text: div.textContent
        })
      },
      // Extension options
      extensions: [
        new HardBreak(),
        new Mention({
          matcher: {
            char: this.mentionPrefix,
          },
          // a list of all suggested items
          items: async () => {
            await new Promise(resolve => {
              setTimeout(resolve, 500)
            })
            return this.mentionItems
          },
          // is called when a suggestion starts
          onEnter: ({
            items, query, range, command, virtualNode,
          }) => {
            this.query = query
            this.filteredItems = items
            this.suggestionRange = range
            this.renderPopup(virtualNode)
            // we save the command for inserting a selected mention
            // this allows us to call it inside of our custom popup
            // via keyboard navigation and on click
            this.insertMention = command
          },
          // is called when a suggestion has changed
          onChange: ({
            items, query, range, virtualNode,
          }) => {
            this.query = query
            this.filteredItems = items
            this.suggestionRange = range
            this.navigatedItemIndex = 0
            this.renderPopup(virtualNode)
          },
          // is called when a suggestion is cancelled
          onExit: () => {
            // reset all saved values
            this.query = null
            this.filteredItems = []
            this.suggestionRange = null
            this.navigatedItemIndex = 0
            this.destroyPopup()
          },
          // is called on every keyDown event while a suggestion is active
          onKeyDown: ({ event }) => {
            this.$emit('mentioning', this.query)
            if (event.key === 'ArrowUp') {
              this.upHandler()
              return true
            }
            if (event.key === 'ArrowDown') {
              this.downHandler()
              return true
            }
            if (event.key === 'Enter') {
              this.enterHandler()
              return true
            }
            return false
          },
        }),
      ],
    })
    this.editor.setContent(this.value)
  },
  computed: {
    hasResults() {
      return this.filteredItems.length
    },
    showSuggestions() {
      return this.query || this.hasResults
    },
  },
  watch: {
    value (val) {
      // so cursor doesn't jump to start on typing
    if (this.editor && val !== this.value) {
        this.editor.setContent(val, true)
      }
    }
  },
  methods: {
    // navigate to the previous item
    // if it's the first item, navigate to the last one
    upHandler() {
      this.navigatedItemIndex = ((this.navigatedItemIndex + this.filteredItems.length) - 1) % this.filteredItems.length
    },
    // navigate to the next item
    // if it's the last item, navigate to the first one
    downHandler() {
      this.navigatedItemIndex = (this.navigatedItemIndex + 1) % this.filteredItems.length
    },
    enterHandler() {
      const item = this.filteredItems[this.navigatedItemIndex]
      if (item) {
        this.selectItem(item)
      }
    },
    // we have to replace our suggestion text with a mention
    // so it's important to pass also the position of your suggestion text
    selectItem(item) {
      this.insertMention({
        range: this.suggestionRange,
        attrs: {
          id: item[this.mentionValue],
          label: item[this.mentionLabel] + this.mentionPostfix,
        },
      })
      this.$emit('mention', item)
      this.editor.focus()
    },
    // renders a popup with suggestions
    // tiptap provides a virtualNode object for using popper.js (or tippy.js) for popups
    renderPopup(node) {
      if (this.popup) {
        return
      }
      // ref: https://atomiks.github.io/tippyjs/v6/all-props/
      this.popup = tippy('.editor', {
        getReferenceClientRect: node.getBoundingClientRect,
        appendTo: () => document.body,
        interactive: true,
        sticky: true, // make sure position of tippy is updated when content changes
        plugins: [sticky],
        content: this.$refs.suggestions,
        trigger: 'mouseenter', // manual
        showOnCreate: true,
        theme: 'dark',
        placement: 'top-start',
        inertia: true,
        duration: [400, 200],
      })
    },
    destroyPopup() {
      if (this.popup) {
        this.popup[0].destroy()
        this.popup = null
      }
    },
  },
  beforeDestroy() {
    this.destroyPopup()
  },
}
</script>
<style>
.mentionable-editor__wrapper .ProseMirror {
  font-family: Arial;
  padding: 10px;
  box-sizing: border-box;
  border: 1px solid rgb(201, 201, 201);
  height: 100%;
  overflow: auto;
}

/* width */
.mentionable-editor__wrapper .ProseMirror::-webkit-scrollbar {
  width: 6px;
}
/* Track */
.mentionable-editor__wrapper .ProseMirror::-webkit-scrollbar-track {
  background: rgb(245, 245, 245);
}
/* Handle */
.mentionable-editor__wrapper .ProseMirror::-webkit-scrollbar-thumb {
  background-color: rgb(230, 230, 230);
  transition-duration: 0.5s;
  border-radius: 4px;
}
/* Handle on hover */
.mentionable-editor__wrapper .ProseMirror::-webkit-scrollbar-thumb:hover {
  background-color: rgb(189, 189, 189);
  cursor: pointer;
}

.mentionable-editor__wrapper .ProseMirror p {
  margin-top: 0;
  margin-bottom: 0.5rem;
}
.mentionable-editor__wrapper .ProseMirror-focused {
  outline: none;
}
.mentionable-editor__wrapper {
  height: 100%;
}
.mentionable-editor__wrapper .editor {
  position: relative;
  height: 100%;
}
.mentionable-editor__wrapper .editor__content {
  height: 100%;
}
.mentionable-editor__wrapper .mention {
  background: rgba(0, 0, 0, 0.1);
  color: rgba(0, 0, 0, 0.6);
  font-size: 0.8rem;
  font-weight: bold;
  border-radius: 5px;
  padding: 0.2rem 0.5rem;
  white-space: nowrap;
}
.mentionable-editor__wrapper .mention-suggestion {
  color: rgba(0, 0, 0, 0.6);
}
.suggestion-list{
  padding: 0.2rem;
  border: 2px solid rgba(0, 0, 0, 0.1);
  font-size: 0.8rem;
  font-weight: bold;
  background-color: rgba(0, 0, 0, 0.9);
}
.suggestion-list__no-results {
  padding: 0.2rem 0.5rem;
}
.suggestion-list__item {
  border-radius: 5px;
  padding: 0.2rem 0.5rem;
  margin-bottom: 0.2rem;
  cursor: pointer;
  color: rgb(255, 255, 255);
  font-family: Arial;
}
.suggestion-list__item:last-child {
  margin-bottom: 0;
}
.suggestion-list__item.is-selected,
.suggestion-list__item:hover {
  background-color: rgba(255, 255, 255, 0.2);
}
.suggestion-list__item.is-empty {
  opacity: 0.5;
}
</style>