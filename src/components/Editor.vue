<template>
  <div class="mentionable-editor__wrapper">
    <div class="editor">
      <editor-content class="editor__content" :editor="editor" />
      <div class="suggestion-list" v-show="showSuggestions" ref="suggestions">
        <template v-if="hasResults">
          <div
            v-for="(user, index) in filteredUsers"
            :key="user.id"
            class="suggestion-list__item"
            :class="{ 'is-selected': navigatedUserIndex === index }"
            @click="selectUser(user)"
          >
            {{ user.name }}
          </div>
        </template>
        <div v-else class="suggestion-list__item is-empty">
          No items found
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Fuse from 'fuse.js'
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
      filteredUsers: [],
      navigatedUserIndex: 0,
      insertMention: () => {},
    }
  },
  mounted() {
    this.editor = new Editor({
      // Editor options
      onUpdate: ({ getHTML }) => {
        this.$emit('input', getHTML())
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
            this.filteredUsers = items
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
            this.filteredUsers = items
            this.suggestionRange = range
            this.navigatedUserIndex = 0
            this.renderPopup(virtualNode)
          },
          // is called when a suggestion is cancelled
          onExit: () => {
            // reset all saved values
            this.query = null
            this.filteredUsers = []
            this.suggestionRange = null
            this.navigatedUserIndex = 0
            this.destroyPopup()
          },
          // is called on every keyDown event while a suggestion is active
          onKeyDown: ({ event }) => {
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
          // is called when a suggestion has changed
          // this function is optional because there is basic filtering built-in
          // you can overwrite it if you prefer your own filtering
          // in this example we use fuse.js with support for fuzzy search
          onFilter: async (items, query) => {
            if (!query) {
              return items
            }
            await new Promise(resolve => {
              setTimeout(resolve, 500)
            })
            const fuse = new Fuse(items, {
              threshold: 0.2,
              keys: ['name'],
            })
            return fuse.search(query).map(item => item.item)
          },
        }),
      ],
    })
    this.editor.setContent(this.value)
  },
  computed: {
    hasResults() {
      return this.filteredUsers.length
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
      this.navigatedUserIndex = ((this.navigatedUserIndex + this.filteredUsers.length) - 1) % this.filteredUsers.length
    },
    // navigate to the next item
    // if it's the last item, navigate to the first one
    downHandler() {
      this.navigatedUserIndex = (this.navigatedUserIndex + 1) % this.filteredUsers.length
    },
    enterHandler() {
      const user = this.filteredUsers[this.navigatedUserIndex]
      if (user) {
        this.selectUser(user)
      }
    },
    // we have to replace our suggestion text with a mention
    // so it's important to pass also the position of your suggestion text
    selectUser(user) {
      this.insertMention({
        range: this.suggestionRange,
        attrs: {
          id: user.id,
          label: user.name + this.mentionPostfix,
        },
      })
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
  border: 1px solid rgb(201, 201, 201);
}
.mentionable-editor__wrapper .ProseMirror p {
  margin-top: 0;
  margin-bottom: 0.5rem;
}
.mentionable-editor__wrapper .ProseMirror-focused {
  outline: none;
}
.mentionable-editor__wrapper .editor {
  position: relative;
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