<template>
    <div id="app">
        <!--<OCImg />-->
        <OCVideo />
    </div>
</template>

<script>
    // import OCImg from './components/OCImg.vue';
    import OCVideo from './components/OCVideo.vue';
    export default {
        name: 'App',
        components: {
            // OCImg,
            OCVideo,
        },
    };
</script>

<style lang="scss">
    #app {
        font-family: Avenir, Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        color: #2c3e50;
    }
</style>
