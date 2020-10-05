<template>
  <div id="app">
    <!--  -->
    <div>
      <Editor
        v-model="content"
        mentionPrefix="$"
        mentionPostfix="$" 
        :mentionItems="[
          { id: 1, name: 'Item 1' },
          { id: 2, name: 'Item 2' },
          { id: 3, name: 'Item 3' },
        ]"
      />
    </div>
    <!--  -->
  </div>
</template>

<script>
import Editor from './components/Editor'
export default {
  components: {
    Editor,
  },
  data() {
    return {
      content: ''
    }
  },
}
</script>
