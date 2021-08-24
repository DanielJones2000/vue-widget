<template>
    <div>
        <TimeAxis :timeAxisList="timeAxisList"></TimeAxis>
    </div>
</template>
<script>
import TimeAxis from './components/time-axis/index.vue'

export default {
    components: {
        TimeAxis
    },
    data(){
        return {
            timeAxisList:[]
        }
    }
}

</script>
<style>
#app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
}
</style>
