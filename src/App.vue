<script setup>
import HelloWorld from './components/HelloWorld.vue'
import TheWelcome from './components/TheWelcome.vue'
</script>

<template>
  <div id="app">
    <Chat />
  </div>
</template>

<script>
import Chat from './components/Chat.vue';

export default {
  name: 'App',
  components: {
    Chat
  }
};
</script>

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }
}

