<script setup>
import { onMounted, ref, watch, computed } from 'vue';

const mapRef = ref(null)
const lngRef = ref(105.8522308815123)
const latRef = ref(21.027864986832427)
const doCaoRef = ref(100)
const rollRef = ref(0)
const pitchRef = ref(0)
const yallRef = ref(180)
const modelRef = ref(null)

const configRef = ref("manual")

const arrLogPlane = ref([])
const showPlayLogUAV = ref(false)
const lineRef = ref(null)

const currentIndex = ref(0);
const isPlaying = ref(false);
const speed = ref(4); // Tốc độ phát (1x, 2x, 4x...)
const interval = ref(null);
const currentTimeMs = computed(() => arrLogPlane.value[currentIndex.value]?.time_ms || 0);
const totalTimeMs = computed(() => arrLogPlane.value[arrLogPlane.value.length - 1]?.time_ms || 1);
const modelPlaneLogRef = ref(null)
const isFocusPlane = ref(true)

// Hàm bắt đầu playback
const startPlayback = () => {
  isPlaying.value = true;
  interval.value = setInterval(() => {
    if (currentIndex.value < arrLogPlane.value.length - 1) {
      currentIndex.value++;
      onAddModelAirplane();
      onViewMapByPlane();
    } else {
      stopPlayback();
    }
  }, 1000 / speed.value);
};
// Hàm dừng playback
const stopPlayback = () => {
  isPlaying.value = false;
  clearInterval(interval.value);
};
// Hàm tua tới (next frame)
const nextFrame = () => {
  if (currentIndex.value < arrLogPlane.value.length - 1) {
    currentIndex.value++;
    onAddModelAirplane();
    onViewMapByPlane();
  }
};

// Hàm tua lui (previous frame)
const prevFrame = () => {
  if (currentIndex.value > 0) {
    currentIndex.value--;
    onAddModelAirplane();
    onViewMapByPlane();
  }
};
// Khi người dùng thay đổi vị trí slider
watch(currentIndex, () => {
  if (currentIndex.value >= arrLogPlane.value.length) {
    stopPlayback();
  }
});

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
                model.setRotation({ x: pitch + 90, y: yaw, z: roll});
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
    showPlayLogUAV.value = false;
    reader.onload = function(event){
        const lines = event.target.result.split(/\r?\n/);
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
        showPlayLogUAV.value = true;
        onAddLine();
        onAddModelAirplane();
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

function onAddLayerPlaneLogByConfig(lng, lat, doCao, roll, pitch, yaw){
    mapRef.value.addLayer({
        id: 'custom-model-plane-log',
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
                let xTransform = pitch + 90;
                let yTransform = Math.abs(yaw) - 5
                let zTransform = roll; 
                model.setRotation({ x: xTransform, y: yTransform, z: zTransform});
                modelPlaneLogRef.value = model;
                tb.add(model);
            });
        },

        render: function () {
            tb.update();
        }
    });
}

function onAddLine(){
    if(lineRef.value != null){
        tb.remove(lineRef.value);
    }
    let coordinates = []
    for(let i = 0; i < arrLogPlane.value.length; i++){
        let item = arrLogPlane.value[i];
        let coordinate = [item['lng'], item['lat'], item['elevator']]
        coordinates.push(coordinate);
    }
    let lineModel = tb.line({
        geometry: coordinates,
        width: 5,
		color: 'red'
    })
    lineRef.value = lineModel;
    tb.add(lineModel);
    mapRef.value.flyTo({
        center: [arrLogPlane.value[0]['lng'], arrLogPlane.value[0]['lat']]
    })
}

function onAddModelAirplane(){
    if(mapRef.value.getLayer('custom-model-plane-log')){
        mapRef.value.removeLayer('custom-model-plane-log');
        tb.remove(modelPlaneLogRef.value);
    }
    let itemActive = arrLogPlane.value[currentIndex.value];
    onAddLayerPlaneLogByConfig(itemActive['lng'], itemActive['lat'], itemActive['elevator'], itemActive['roll'], itemActive['pitch'], itemActive['yaw'],);
}

function onViewMapByPlane(){
    if(isFocusPlane.value){
        if(arrLogPlane.value[currentIndex.value - 50] != null && arrLogPlane.value[currentIndex.value + 250] != null){
            let coordinate_last = [arrLogPlane.value[currentIndex.value - 50]['lng'], arrLogPlane.value[currentIndex.value - 50]['lat']];
            let coordinate_next = [arrLogPlane.value[currentIndex.value + 250]['lng'], arrLogPlane.value[currentIndex.value + 250]['lat']];
            let coordinateActive = [arrLogPlane.value[currentIndex.value]['lng'], arrLogPlane.value[currentIndex.value]['lat']];
            var bearing = turf.bearing(
                turf.point(coordinate_last),
                turf.point(coordinate_next)
            )
            mapRef.value.panTo(coordinateActive, { animate: true, essential: true, curve: 1.42, duration: 1000 / speed.value, pitch: 45, bearing: bearing })
        }
        
    }
}

</script>

<template>
<div style="width: 100vw;height:100vh; position: relative;" id="map">
    <div style="width: 320px; position: absolute;top: 10px;left: 25px; z-index: 10;background-color: #fff; padding: 20px;">
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
            <div class="playback-container" v-show="showPlayLogUAV">
                    <!-- Thanh thời gian -->
                    <div class="timeline">
                    <span>{{ currentTimeMs }} ms</span>
                    <input type="range" min="0" :max="arrLogPlane.length - 1" v-model="currentIndex" />
                    <span>{{ totalTimeMs }} ms</span>
                    </div>

                    <!-- Nút điều khiển -->
                    <div class="controls">
                        <button @click="prevFrame">⏮</button>
                        <button @click="isPlaying ? stopPlayback() : startPlayback()">
                            {{ isPlaying ? "⏸ Pause" : "▶ Play" }}
                        </button>
                        <button @click="nextFrame">⏭</button>
                        <div style="display: flex;align-items: center;">
                            <label>
                                Speed:
                                <select v-model="speed">
                                <option :value="1">1x</option>
                                <option :value="2">2x</option>
                                <option :value="4">4x</option>
                                <option :value="6">6x</option>
                                <option :value="8">8x</option>
                                <option :value="12">12x</option>
                                <option :value="16">16x</option>
                                </select>
                            </label>
                            <div style="display: flex;align-items: center;margin-left: 6px;">
                                <input type="checkbox" id="myCheckbox" name="myCheckbox" v-model="isFocusPlane"> Focus
                            </div>
                        </div>
                        
                    </div>

                    <!-- Hiển thị thông tin UAV -->
                    <div class="info">
                        <p><strong>Time:</strong> {{ currentTimeMs }} ms</p>
                        <p><strong>Lng:</strong> {{ arrLogPlane[currentIndex]?.lng }}</p>
                        <p><strong>Lat:</strong> {{ arrLogPlane[currentIndex]?.lat }}</p>
                        <p><strong>elevator:</strong> {{ arrLogPlane[currentIndex]?.elevator }}</p>
                        <p><strong>Yaw:</strong> {{ arrLogPlane[currentIndex]?.yaw }}</p>
                        <p><strong>Roll:</strong> {{ arrLogPlane[currentIndex]?.roll }}</p>
                        <p><strong>Pitch:</strong> {{ arrLogPlane[currentIndex]?.pitch }}</p>
                    </div>
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
.playback-container {
  text-align: center;
  margin-top: 15px;
}

.timeline {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 10px;
}

.controls {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-bottom: 20px;
}

button {
  background: #ff9800;
  border: none;
  padding: 5px 10px;
  border-radius: 5px;
  cursor: pointer;
  color: white;
  font-size: 14px;
}

button:hover {
  background: #e68900;
}

input[type="range"] {
  width: 70%;
}

.info {
  background: #292929;
  padding: 10px;
  border-radius: 5px;
  text-align: left;
  font-size: 14px;
  color: #fff;
}

.info p {
  margin: 5px 0;
}
</style>
