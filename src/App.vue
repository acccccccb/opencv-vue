<template>
    <div id="app">
        <OpenCvJs />
    </div>
</template>

<script>
    import OpenCvJs from './components/OpenCvJs.vue';
    export default {
        name: 'App',
        components: {
            OpenCvJs,
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
