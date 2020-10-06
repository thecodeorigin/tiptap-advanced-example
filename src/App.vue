<template>
  <div id="app">
    <h1>Try to tag with '$' and paste image into the textarea</h1>
    <!--  -->
    <div style="width: 300px; height: 100px">
      <MentionableEditor
        id="mentionable-editor"
        v-model="content"
        @mentioning="onMentioning"
        @mention="onMention"
        @change="onChange"
        @paste="onPaste"
        :mentionItems="filteredData"
        mentionPrefix="$"
        mentionPostfix="$"
      >
        <!-- Slot and scope example, this component has a default slot just like this -->
        <template v-slot:mention-item="{item}">
          {{ item.name }}
        </template>
        <template v-slot:no-results-item>
          No items found
        </template>
        <!-- You can customize suggestion list by these classes -->
        <!-- .mentionable-editor__wrapper .mention -->
        <!-- .mentionable-editor__wrapper .mention-suggestion -->
        <!-- .suggestion-list -->
        <!-- .suggestion-list__no-results -->
        <!-- .suggestion-list__item -->
        <!-- .suggestion-list__item .is-selected -->
        <!-- .suggestion-list__item .is-empty -->
        <!-- .suggestion-list__item:last-child -->
        <!-- .suggestion-list__item:hover -->
        <!-- And so on -->
      </MentionableEditor>
    </div>
    <div>
      <img style="width: 100px" :src="image" v-for="(image, index) in imageList" :key="'pasted-image-' + index">
    </div>
    <!--  -->
  </div>
</template>

<script>
import MentionableEditor from './components/MentionableEditor'
export default {
  components: {
    MentionableEditor,
  },
  data() {
    return {
      content: '',
      mentioning: '',
      data: [
        { id: 1, name: 'NAME' },
        { id: 2, name: 'BIRTHDAY' },
        { id: 3, name: 'SOMETHING' },
        { id: 4, name: 'AH_YES' },
      ],
      imageList: []
    }
  },
  computed: {
    filteredData() {
      return this.data.filter( item => item.name.includes(this.mentioning))
    }
  },
  methods: {
    onMentioning(payload) {
      this.mentioning = payload
    },
    onMention(payload) {
      console.log(payload)
    },
    onChange(payload) {
      console.log(payload)
    },
    onPaste(payload) {
      if(payload.type === 'image') {
        this.imageList.push(payload.value)
      }
    }
  }
}
</script>
