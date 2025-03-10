<script setup>
import { onMounted, ref } from 'vue';

const mapRef = ref(null)
const lngRef = ref(105.8522308815123)
const latRef = ref(21.027864986832427)
const doCaoRef = ref(100)
const rollRef = ref(0)
const pitchRef = ref(90)
const yallRef = ref(180)
const modelRef = ref(null)

const configRef = ref("manual")

const arrLogPlane = ref([])
const disableBtnMoPhong = ref(true)
const lineRef = ref(null)

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
                obj: '/file-1571059202690.glb', //./public/file-1571059202690.glb
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

function onChangeFileLog(evt){
    let file = evt.target.files[0];
    const reader = new FileReader();
    disableBtnMoPhong.value = true;
    reader.onload = function(event){
        const lines = event.target.result.split(/\r?\n/);
        let arrByte = []
        lines.forEach((line, index) => {
            if(line.includes("*T02:")){
                const match = line.match(/\*T02:\s*(.*)/);
                if(match){
                    let base64Str = match[1];
                    const byteArray = base64ToBytes(base64Str);
                    const time_ms = bytesToInt32(byteArray, 0);
                    const elevator_scale = bytesToInt32(byteArray, 4);
                    const lat_scale = bytesToInt32(byteArray, 8);
                    const long_scale = bytesToInt32(byteArray, 12);
                    const yaw_rad_scale = byteToInt16(byteArray, 16);
                    const roll_rad_scale = byteToInt16(byteArray, 18);
                    const pitch_rad_scale = byteToInt16(byteArray, 20);
                    const elevator = elevator_scale/100;
                    const lat = lat_scale/1000000;
                    const lon = long_scale/1000000;
                    const yaw_rad = yaw_rad_scale/1000;
                    const roll_rad = roll_rad_scale/1000;
                    const pitch_rad = pitch_rad_scale/1000;
                    if(elevator > 0 && lon > 0 && lat > 0){
                        let item = {
                            'time_ms': time_ms,
                            'lng': lon,
                            'lat': lat,
                            'elevator': elevator,
                            'yaw': radianToDegree(yaw_rad),
                            'roll': radianToDegree(roll_rad),
                            'pitch': radianToDegree(pitch_rad)
                        }
                        arrLogPlane.value.push(item);
                    }
                }
            }
        });
        disableBtnMoPhong.value = false;
    }
    reader.readAsText(file);
}

function base64ToBytes(base64){
    const binaryString = atob(base64);
    const len = binaryString.length;
    const bytes = new Uint8Array(len);
    for(let i = 0; i < len; i++){
        bytes[i] = binaryString.charCodeAt(i);
    }
    return bytes;
}

function bytesToInt32(byteArray, index, littleEndian = true){
    const view = new DataView(byteArray.buffer);
    return littleEndian ? view.getInt32(index, true) : view.getInt32(index, false);
}

function byteToInt16(byteArray, index, littleEndian = true){
    const view = new DataView(byteArray.buffer);
    return littleEndian ? view.getInt16(index, true) : view.getInt16(index, false);
}

function radianToDegree(radian) {
    return radian * (180 / Math.PI);
}


function onApplyFileLog(){
    console.log("Bắt đầu mô phỏng");
    console.log(arrLogPlane.value);
    if(mapRef.value.getLayer('custom-threebox-model')){
        mapRef.value.removeLayer('custom-threebox-model');
        tb.remove(modelRef.value);
    }
    addLayerLinePlane();
}

function addLayerLinePlane(){
    let flightPlane = {
        'type': "Feature",
        'geometry': {
            'coordinates': []
        },
        'properties': {}
    }
    let arrLineTurf = [];
    for(let i = 0; i < arrLogPlane.value.length; i++){
        let item = arrLogPlane.value[i];
        flightPlane.geometry.coordinates.push([item['lng'], item['lat'], item['elevator']])
        arrLineTurf.push([item['lng'], item['lat']])
    }
    var lineTurf = turf.lineString(arrLineTurf);
    var bbox = turf.bbox(lineTurf);
    console.log(flightPlane)
    lineRef.value = tb.line({
        geometry: flightPlane.geometry.coordinates,
        width: 50,
        color: 'red'
    })
    tb.add(lineRef.value);
    mapRef.value.fitBounds(bbox)
}
</script>

<template>
<div style="width: 100vw;height:100vh; position: relative;" id="map">
    <div style="width: 300px; position: absolute;top: 10px;left: 25px; z-index: 10;background-color: #fff; padding: 20px;">
        <label class="label_t">Chế độ nhập dữ liệu</label><br>
        <div class="type_display">
            <input type="radio" id="manual" name="config" value="manual" v-model="configRef">
            <label for="manual">Thủ công</label>
            <input type="radio" id="file_log" name="config" value="file_log" style="margin-left: 10px;" v-model="configRef">
            <label for="file_log">Phát lại từ Log</label>
        </div>
        <template v-if="configRef == 'manual'">
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
        </template>
        <template v-if="configRef == 'file_log'">
            <label class="label_t">Chọn tệp nhật ký bay</label>
            <input type="file" id="log" name="log" accept=".txt" @change="onChangeFileLog"/>
            <div style="width: 100%;margin-top: 15px;display: flex;justify-content: end;">
                <button @click="onApplyFileLog()" :disabled="disableBtnMoPhong">Mô phỏng</button>
            </div>
        </template>
        
    </div>
</div>
</template>

<style scoped>
.label_t{
    font-size: 15px;
    font-weight: 600;
}
.type_display{
    width: 100%;
    display: flex;
    margin-bottom: 10px;
    align-items: center;
}
</style>
