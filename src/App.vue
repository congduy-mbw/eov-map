<script setup>
import HelloWorld from './components/HelloWorld.vue'
import { onMounted, ref } from 'vue';

const mapRef = ref(null)
const lngRef = ref(105.8522308815123)
const latRef = ref(21.027864986832427)
const doCaoRef = ref(100)
const rollRef = ref(0)
const pitchRef = ref(90)
const yallRef = ref(180)
const modelRef = ref(null)

onMounted(() => {
  mapboxgl.accessToken = 'pk.eyJ1IjoiZHV5ZGMiLCJhIjoiY2t2djUxZTl3MDFrNDJvbHU2eXd1YmkwdyJ9.EJMYZxWRlMshYpLlSEOmmg';
  mapRef.value =  new mapboxgl.Map({
    container: 'map',
    style: 'mapbox://styles/mapbox/satellite-v9',
    zoom: 17.74,
    center: [105.852286, 21.029583],
    hash: true,
    pitch: 64.9,
  });
  mapRef.value.addControl(new mapboxgl.NavigationControl());
  const tb = (window.tb = new Threebox(
    mapRef.value,
    mapRef.value.getCanvas().getContext('webgl'),
        {
            defaultLights: true
        }
    ));

    mapRef.value.on('style.load', () => {
        onAddLayerByConfig(lngRef.value, latRef.value, doCaoRef.value, rollRef.value, pitchRef.value, yallRef.value)
  });
})

function onAddLayerByConfig(lng, lat, doCao, roll, pitch, yaw){
    mapRef.value.addLayer({
        id: 'custom-threebox-model',
        type: 'custom',
        renderingMode: '3d',
        onAdd: function () {
            const options = {
                obj: './public/file-1571059202690.glb', //
                type: 'gltf',
                scale: 50,
                units: 'meters',
                anchor: 'center',
                bbox: false
            };

            tb.loadObj(options, (model) => {
                model.setCoords([lng, lat, doCao]);
                model.setRotation({ x: pitch, y: yaw, z: roll});
                modelRef.value = model;
                tb.add(model);
            });
        },

        render: function () {
            tb.update();
        }
    });
}

function onApplyConfig(){
    let lng = parseFloat(lngRef.value)
    let lat = parseFloat(latRef.value)
    let doCao = parseFloat(doCaoRef.value)
    let roll = parseFloat(rollRef.value)
    let pitch = parseFloat(pitchRef.value)
    let yall = parseFloat(yallRef.value)
    if(mapRef.value.getLayer('custom-threebox-model')){
        mapRef.value.removeLayer('custom-threebox-model');
        tb.remove(modelRef.value);
        mapRef.value.flyTo({
            center: [lng, lat]
        })
        onAddLayerByConfig(lng, lat, doCao, roll, pitch, yall);
    }

}
</script>

<template>
<div style="width: 100vw;height:100vh; position: relative;" id="map">
    <div style="width: 300px; position: absolute;top: 10px;left: 25px; z-index: 10;background-color: #fff; padding: 20px;">
        <label class="label_t">Kinh độ</label><br>
        <input type="number" id="longitude" style="width:100%;margin-bottom: 10px;" v-model="lngRef"/><br>
        <label class="label_t">Vĩ độ</label><br>
        <input type="number" id="latitude" style="width:100%;margin-bottom: 10px;" v-model="latRef"/><br>
        <label class="label_t">Độ cao(m)</label><br>
        <input type="number" id="docao" style="width:100%;margin-bottom: 10px;" v-model="doCaoRef"/><br>
        <label class="label_t">Trạng thái quay(Roll, Pitch, Yaw)</label><br>
        <label>Roll</label><br>
        <input type="number" id="roll" placeholder="Roll" style="width:100%;margin-bottom: 5px;"  v-model="rollRef"/><br>
        <label>Pitch</label><br>
        <input type="number" id="pitch" placeholder="Pitch" style="width:100%;margin-bottom: 5px;"  v-model="pitchRef"/><br>
        <label>Yaw</label><br>
        <input type="number" id="yall" placeholder="Yaw" style="width:100%;margin-bottom: 5px;"  v-model="yallRef"/><br>

        <div style="width: 100%;margin-top: 15px;display: flex;justify-content: end;">
            <button @click="onApplyConfig()">Áp dụng</button>
        </div>
        
    </div>
</div>
</template>

<style scoped>
.label_t{
    font-size: 15px;
    font-weight: 600;
}
</style>
