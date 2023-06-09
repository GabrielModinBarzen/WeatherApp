<script setup>
    import axios from 'axios';
    import {ref} from 'vue';
    const emit = defineEmits(["weather"])

    let previousLocation = null;
    let previousTime = null;
    let enabled = ref(true);
    let intervalID;
    let currentWeatherObj = null;
    tickWeatherApi();
    toggleTicker();
    function toggleTicker() {
        if (enabled.value == true) {
            intervalID = window.setInterval(tickWeatherApi,10000);
        } else {
            window.clearInterval(intervalID);
        }
    }
    
    function tickWeatherApi() {
        navigator.geolocation.getCurrentPosition(
            (success) => checkWeatherApiRequest(success), 
            (error) => geoLocationError(error));   
    }

    function checkWeatherApiRequest(success) {
        console.log("=RUNNING CHECK BEFORE CALL API=")
        let currentTime = Date.now();
        let currentLocation = success;
        previousLocation = success;
        if (previousTime === null || previousLocation === null) {
            previousLocation = currentLocation;
            makeApiRequest(currentLocation.coords.latitude,currentLocation.coords.longitude);
            return;
        } else if ((currentTime - previousTime) > 60000) {
            previousLocation = currentLocation;
            makeApiRequest(currentLocation.coords.latitude,currentLocation.coords.longitude);
            return;
        } else {
            var previousCoord = previousLocation.coords;
            var currentCoord = currentLocation.coords;
            var distanceInKm = getDistanceFromLatLonInKm(
                previousCoord.latitude,
                previousCoord.longitude,
                currentCoord.latitude,
                currentCoord.longitude)
            if (distanceInKm > 0.5) {
                previousLocation = currentLocation;
                makeApiRequest(currentLocation.coords.latitude,currentLocation.coords.longitude);
                return;
            }
        }
        console.log("=CONDITIONS NOT MET, MOVE OR WAIT=")
    }

    function makeApiRequest(currentLatitude, currentLongitude) {
        //External API only takes 6 decimals for lat and lon parameters
        var lon = currentLongitude.toFixed(6);
        var lat = currentLatitude.toFixed(6);

        axios({
            method: "get",
            url: `https://opendata-download-metfcst.smhi.se/api/category/pmp3g/version/2/geotype/point/lon/${lon}/lat/${lat}/data.json`
        })
        .catch((error) => apiRequestError(error))
        .then((data) => apiRequestSuccess(data));
    }
    function geoLocationError(error) {
        let errorData = {}
        errorData.message = error;
        console.log(errorData);
    }

    function apiRequestError(error) {
        let errorData = {}
        errorData.message = error;
        emit('weather', errorData);
    }

    function getDistanceFromLatLonInKm(latitudeFrom,longitudeFrom,latitudeTo,longitudeTo) {
      var R = 6371; // Radius of the earth in km
      var distanceLatitude = degreeToRadian(latitudeTo-latitudeFrom);  // deg2rad below
      var distanceLongitude = degreeToRadian(longitudeTo-longitudeFrom); 
      var a = 
        Math.sin(distanceLatitude/2) * Math.sin(distanceLatitude/2) +
        Math.cos(degreeToRadian(latitudeFrom)) * Math.cos(degreeToRadian(latitudeTo)) * 
        Math.sin(distanceLongitude/2) * Math.sin(distanceLongitude/2); 
      var c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a)); 
      var d = R * c; // Distance in km
      return d;
    }

    function degreeToRadian(degree) {
      return degree * (Math.PI/180)
    }
    function apiRequestSuccess(data) {
        previousTime = Date.now();
        let jsonData = JSON.parse(data.request.response);
        //let now = jsonData.timeSeries[2];
        const timeWithinHour = jsonData.timeSeries.filter((item) => {
            return Date.parse(item.validTime) - Date.now() < 6000000
        })
        const currentTimeParameters = timeWithinHour[timeWithinHour.length-1]
        processTimeParameters(currentTimeParameters);
    }

    function processTimeParameters(currentTimeParameters) {
        let tempData = currentTimeParameters.parameters[10];
        let airPressureData = currentTimeParameters.parameters[11];
        let weatherSymbolIntegerData = currentTimeParameters.parameters[18];
        let windSpeedData = currentTimeParameters.parameters[14];
        let windDirectionData = currentTimeParameters.parameters[13];
        let percipitationCategoryData = currentTimeParameters.parameters[1];
        let horizontalVisibilityData = currentTimeParameters.parameters[12];
        let frozenPercipitationPercentageData = currentTimeParameters.parameters[0];

        let weather = {};
        let temperature = {value:tempData.values[0] , unit:tempData.unit, name:"Temperature" };
        let airPressure = {value:airPressureData.values[0] , unit:airPressureData.unit, name:"Air pressure" };
        let weatherSymbolInteger = {value:weatherSymbolIntegerData.values[0] , unit:weatherSymbolIntegerData.unit ,name:"Weather symbol"};
        let windSpeed = {value:windSpeedData.values[0] , unit:windSpeedData.unit, name:"Wind speed"};
        let windDirection = {value:windDirectionData.values[0] , unit:windDirectionData.unit, name:"Wind direction" };
        let percipitationCategory = {value:percipitationCategoryData.values[0],unit:percipitationCategoryData.unit, name:"Weather category"};
        let horizontalVisibility = {value:horizontalVisibilityData.values[0], unit:horizontalVisibilityData.unit, name:"Horizontal visibility"};
        let frozenPercipitationPercentage = {value:frozenPercipitationPercentageData.values[0], unit:frozenPercipitationPercentageData.unit, name:"Hail percentage"};

        weather.temperature = temperature;
        weather.airPressure = airPressure;
        weather.weatherSymbolInteger = weatherSymbolInteger;
        weather.windSpeed = windSpeed;
        weather.windDirection = windDirection;
        weather.percipitationCategory = percipitationCategory;
        weather.horizontalVisibility = horizontalVisibility;
        weather.frozenPercipitationPercentage = frozenPercipitationPercentage
        weather.time = currentTimeParameters.validTime;

        currentWeatherObj = weather;
        

        //printLocalStorageSize()
        
        emit('weather',currentWeatherObj);
    }



    function printLocalStorageSize() {
        var _lsTotal = 0,_xLen,_x;for(_x in localStorage){ if(!localStorage.hasOwnProperty(_x)){continue;} _xLen= ((localStorage[_x].length + _x.length)* 2);_lsTotal+=_xLen; console.log(_x.substr(0,50)+" = "+ (_xLen/1024).toFixed(2)+" KB")};console.log("Total = " + (_lsTotal / 1024).toFixed(2) + " KB");
        console.log(_lsTotal);
    }

// 0 : Object { name: "spp"     , Percent of precipitation in frozen form
// 1 : Object { name: "pcat"    , Precipitation category Integer
// 2 : Object { name: "pmin"    , Minimum precipitation intensity
// 3 : Object { name: "pmean"   , Mean precipitation intensity
// 4 : Object { name: "pmax"    , Maximum precipitation intensity
// 5 : Object { name: "pmedian" , Median precipitation intensity
// 6 : Object { name: "tcc_mean", Mean value of total cloud cover
// 7 : Object { name: "lcc_mean", Mean value of low level cloud cover
// 8 : Object { name: "mcc_mean", Mean value of mediul level cloud cover
// 9 : Object { name: "hcc_mean", Mean value of high level cloud cover
// 10: Object { name: "t"       , temperature
// 11: Object { name: "msl"     , Air pressure
// 12: Object { name: "vis"     , Horizontal visibility
// 13: Object { name: "wd"      , Wind direction Degree
// 14: Object { name: "ws"      , Wind speed
// 15: Object { name: "r"       , Relative Humidity
// 16: Object { name: "tstm"    , Thunder probability
// 17: Object { name: "gust"    , Wind gust speed, one dcimal
// 18: Object { name: "Wsymb2"  , Weather symbol integer

</script>
<template>
    <!-- <span @click="enabled = !enabled; toggleTicker();" aria-current="page" :class="{ 'text-green-500' : enabled }, { 'text-red-500' : !enabled } " class="text-gray-300 hover:bg-gray-700 hover:text-white rounded-md px-3 py-2 text-sm font-medium"> -->
    <!--         Tracking :  -->
    <!--         <span v-if="enabled">Enabled</span> -->
    <!--         <span v-else>Disabled</span> -->
    <!-- </span> -->
</template>
