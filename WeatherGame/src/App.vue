<script setup>
    import { ref, onMounted } from 'vue'
    import Navigation from "./components/Navigation.vue"
    import Footer from "./components/Footer.vue"
    import ApiTicker from "./components/ApiTicker.vue"
    import AchivementListener from "./components/AchievementListener.vue"

    let tempUnit = localStorage.getItem("tempUnit");
    if (tempUnit == null) {
        localStorage.setItem("tempUnit", "C");
    }


    function updateWeather(weatherData) {
        tempUnit = localStorage.getItem("tempUnit");
        if (tempUnit == null) {
            localStorage.setItem("tempUnit", "C");
        }
        console.log("UPDATING WEATHER");
        localWeatherData.value = weatherData;
        storeWeatherData(weatherData);
    }
    const weatherDataChangeListener = ref(null);
    
    function storeWeatherData(weatherData) {
        let localStorageHistory = {};

        if (localStorage.getItem("weatherHistory") != null) {
            localStorageHistory = JSON.parse(localStorage.getItem("weatherHistory"));
        }
        
        localStorageHistory[weatherData.time] = weatherData;
        localStorage.setItem("weatherHistory",JSON.stringify(localStorageHistory));
        localStorage.setItem("stats",JSON.stringify(weatherData));
        console.log(weatherDataChangeListener.value)
        weatherDataChangeListener.value.updateAchievements();
    }
    let localWeatherData = ref({});

</script>

<template>
    <div class="flex flex-col h-screen">
        <div>
            <Navigation />
        </div>
        <div class="flex-grow">
            <main >
                <router-view :weather="localWeatherData"/>
            </main>
        </div>
        <div class="relative">
            <Footer class="h-16 text-xl"/>

        </div>
    </div>
    <ApiTicker  @weather="updateWeather"></ApiTicker>
    <AchivementListener ref="weatherDataChangeListener"></AchivementListener>
</template>

<style scoped>
</style>
