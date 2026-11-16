<!DOCTYPE html>
<html lang="bn">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<meta name="theme-color" content="#081522">
<title>MAHIR GPS V3.0</title>

<link href="https://unpkg.com/maplibre-gl@5.6.1/dist/maplibre-gl.css" rel="stylesheet">

<style>
*{box-sizing:border-box}
body{margin:0;background:#091522;color:white;font-family:Arial,sans-serif}
header{height:58px;background:#102237;display:flex;align-items:center;justify-content:space-between;padding:0 12px}
.logo{font-weight:900;font-size:19px;color:#38d8ff}
#gpsStatus{font-size:11px;color:#86eeb0}
#map{height:55vh;min-height:310px;width:100%}
.panel{background:#102237;padding:10px}
.search{display:flex;gap:6px}
input,select{background:#253b53;color:white;border:1px solid #46617d;border-radius:9px;padding:12px;min-width:0}
input{flex:1}
button{background:#087fae;color:white;border:0;border-radius:9px;padding:11px 9px;font-weight:bold}
button:active{opacity:.7}
button.secondary{background:#354d68}
button.active{background:#0caa80}
.stats{display:grid;grid-template-columns:repeat(4,1fr);gap:6px;margin-top:9px}
.stat{background:#233a52;border-radius:9px;padding:9px 2px;text-align:center}
.stat strong{display:block;font-size:18px;color:#4bdeff}
.stat small{font-size:9px;color:#d0dce8}
.grid{display:grid;grid-template-columns:repeat(4,1fr);gap:6px}
.grid button{font-size:10px;min-height:42px;padding:7px 2px}
#notice{font-size:12px;color:#b4eaff;padding:8px 2px}
#alert{font-size:12px;color:#ff8585;font-weight:bold}
#nextTurn{background:#075b79;padding:11px;border-radius:10px;margin-top:8px;font-size:13px}
#steps{max-height:90px;overflow:auto;font-size:12px;line-height:1.8}
#savedList{font-size:12px;line-height:2}
footer{text-align:center;color:#8095a8;font-size:10px;padding:10px}
</style>
</head>

<body>

<header>
 <div class="logo">🧭 MAHIR GPS V3</div>
 <div id="gpsStatus">● GPS OFFLINE</div>
</header>

<div class="panel">
 <div class="search">
  <input id="destination" placeholder="Search destination / গন্তব্য">
  <button onclick="searchDestination()">GO</button>
 </div>

 <div class="stats">
  <div class="stat"><strong id="speed">0</strong><small>KM/H</small></div>
  <div class="stat"><strong id="limit">--</strong><small>SPEED LIMIT</small></div>
  <div class="stat"><strong id="distance">--</strong><small>KM LEFT</small></div>
  <div class="stat"><strong id="eta">--</strong><small>ETA</small></div>
 </div>

 <div id="nextTurn">🧭 Navigation Ready</div>
 <div id="notice">GPS চালু করতে Locate চাপুন।</div>
 <div id="alert"></div>
</div>

<div id="map"></div>

<div class="panel">
 <div class="grid">
  <button onclick="locate()">📍 Locate</button>
  <button id="voiceBtn" onclick="toggleVoice()">🔊 Voice ON</button>
  <button onclick="toggle3D()">🏙️ 3D View</button>
  <button onclick="toggleSatellite()">🛰️ Satellite</button>

  <button onclick="loadRoads()">🚦 Road Data</button>
  <button onclick="loadPOI('fuel')">⛽ Fuel</button>
  <button onclick="loadPOI('hospital')">🏥 Hospital</button>
  <button onclick="loadPOI('parking')">🅿 Parking</button>

  <button onclick="loadPOI('restaurant')">🍽️ Food</button>
  <button onclick="loadPOI('place_of_worship')">🕌 Mosque</button>
  <button onclick="loadPOI('atm')">🏧 ATM</button>
  <button onclick="loadPOI('charging_station')">🔋 EV</button>

  <button onclick="savePlace()">⭐ Save Place</button>
  <button onclick="showSaved()">📌 Favorites</button>
  <button onclick="loadHistory()">🕘 History</button>
  <button onclick="changeLimit()">⚙ Speed Limit</button>

  <button onclick="setLanguage('bn-BD')">🇧🇩 বাংলা</button>
  <button onclick="setLanguage('en-US')">🇬🇧 English</button>
  <button onclick="shareLocation()">📤 Share GPS</button>
  <button onclick="clearRoute()">✖ Clear Route</button>
 </div>
 <div id="steps">Route instructions এখানে দেখা যাবে।</div>
 <div id="savedList"></div>
</div>

<footer>
MAHIR GPS V3.0 | Map © OpenStreetMap contributors | Navigation data may be incomplete
</footer>

<script src="https://unpkg.com/maplibre-gl@5.6.1/dist/maplibre-gl.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>

<script>
// =====================================
// CONFIGURATION
// =====================================
const CONFIG={
 SUPABASE_URL:"YOUR_SUPABASE_URL",
 SUPABASE_ANON_KEY:"YOUR_SUPABASE_ANON_KEY",
 API_URL:"/.netlify/functions/api"
};

const $=id=>document.getElementById(id);
const validBackend=CONFIG.SUPABASE_URL.startsWith("https://") &&
 CONFIG.SUPABASE_ANON_KEY!=="YOUR_SUPABASE_ANON_KEY";

let sb=null,accessToken=null;
let gpsWatch=null,position=null,userMarker=null;
let destinationMarker=null,selectedDestination=null;
let route=null,routeSteps=[],routeCoords=[];
let cameraData=[],roadMarkers=[],poiMarkers=[];
let voiceEnabled=true,voiceLanguage='bn-BD';
let speedLimit=50,lastAlert=0,lastInstruction=-1;
let threeD=false,satellite=false;
let rerouting=false;
let savedPlaces=[];
let historyItems=[];
let lastSave=0;

// =====================================
// MAP
// =====================================
const map=new maplibregl.Map({
 container:'map',
 style:'https://tiles.openfreemap.org/styles/liberty',
 center:[46.6753,24.7136],
 zoom:12,
 pitch:0
});

map.addControl(new maplibregl.NavigationControl(),'top-right');
map.addControl(new maplibregl.ScaleControl(),'bottom-left');

map.on('load',()=>{
 $('notice').innerText='Map loaded. GPS চালু করুন।';
});

map.on('error',e=>{
 console.warn('Map error',e.error);
});

// =====================================
// BACKEND AUTH
// =====================================
async function initBackend(){
 if(!validBackend){
  $('notice').innerText='Backend setup বাকি। Map ও GPS ব্যবহার করা যাবে।';
  return;
 }
 try{
  sb=window.supabase.createClient(
   CONFIG.SUPABASE_URL,CONFIG.SUPABASE_ANON_KEY
  );
  let {data,error}=await sb.auth.getSession();
  if(error)throw error;
  if(!data.session){
   let result=await sb.auth.signInAnonymously();
   if(result.error)throw result.error;
   data=result.data;
  }
  accessToken=data.session?.access_token;
  $('notice').innerText='☁️ Secure cloud backend connected';
 }catch(e){
  console.error(e);
  $('notice').innerText='Backend connection failed: '+e.message;
 }
}

async function backend(action,data={}){
 if(!sb || !accessToken)throw Error('Backend not connected');
 const {data:sessionData}=await sb.auth.getSession();
 accessToken=sessionData.session?.access_token;
 if(!accessToken)throw Error('Login session unavailable');

 const response=await fetch(CONFIG.API_URL,{
  method:'POST',
  headers:{
   'Content-Type':'application/json',
   'Authorization':'Bearer '+accessToken
  },
  body:JSON.stringify({action,data})
 });
 const result=await response.json();
 if(!response.ok)throw Error(result.error||'Backend Error');
 return result;
}

// =====================================
// VOICE
// =====================================
function speak(message){
 if(!voiceEnabled || !window.speechSynthesis)return;
 speechSynthesis.cancel();
 const u=new SpeechSynthesisUtterance(message);
 u.lang=voiceLanguage;
 u.rate=.95;
 speechSynthesis.speak(u);
}

function toggleVoice(){
 voiceEnabled=!voiceEnabled;
 $('voiceBtn').innerText=voiceEnabled?'🔊 Voice ON':'🔇 Voice OFF';
 if(voiceEnabled)speak(voiceLanguage==='bn-BD'?
  'মাহির জিপিএস চালু হয়েছে':'Mahir GPS is ready');
}

function setLanguage(lang){
 voiceLanguage=lang;
 $('notice').innerText=lang==='bn-BD'?'বাংলা ভয়েস নির্বাচিত':'English voice selected';
}

// =====================================
// GPS
// =====================================
function locate(){
 if(!navigator.geolocation){
  alert('GPS supported নয়');return;
 }
 if(gpsWatch!==null)navigator.geolocation.clearWatch(gpsWatch);

 $('gpsStatus').innerText='● CONNECTING';

 gpsWatch=navigator.geolocation.watchPosition(p=>{
  const c=p.coords;
  position={lat:c.latitude,lon:c.longitude};
  const kmh=c.speed===null?0:Math.round(c.speed*3.6);

  $('speed').innerText=kmh;
  $('gpsStatus').innerText='● GPS ACTIVE';
  $('notice').innerText='GPS Accuracy: '+Math.round(c.accuracy)+' m | Altitude: '+(c.altitude===null?'--':Math.round(c.altitude)+'m');

  if(!userMarker){
   userMarker=new maplibregl.Marker({color:'#00cfff'})
    .setLngLat([position.lon,position.lat])
    .setPopup(new maplibregl.Popup().setText('Your location'))
    .addTo(map);
   map.flyTo({center:[position.lon,position.lat],zoom:16});
  }else userMarker.setLngLat([position.lon,position.lat]);

  if(kmh>speedLimit){
   $('alert').innerText='⚠️ Speed limit exceeded!';
   if(Date.now()-lastAlert>15000){
    speak(voiceLanguage==='bn-BD'?'সতর্কতা, গতি কমিয়ে দিন':'Warning, reduce your speed');
    lastAlert=Date.now();
   }
  }else $('alert').innerText='';

  if(selectedDestination){
   updateProgress();
  }
  checkCameras();
 },e=>{
  $('gpsStatus').innerText='● GPS ERROR';
  $('notice').innerText='Location permission দিন: '+e.message;
 },{enableHighAccuracy:true,maximumAge:1000,timeout:20000});
}

// =====================================
// DESTINATION SEARCH
// =====================================
async function searchDestination(){
 const q=$('destination').value.trim();
 if(!q)return alert('গন্তব্য লিখুন');

 $('notice').innerText='Searching...';
 try{
  const r=await fetch(
   'https://nominatim.openstreetmap.org/search?format=json&limit=1&q='+encodeURIComponent(q)
  );
  const d=await r.json();
  if(!d.length)throw Error('Place not found');

  selectedDestination={
   lat:+d[0].lat,lon:+d[0].lon,name:d[0].display_name
  };
  if(destinationMarker)destinationMarker.remove();

  destinationMarker=new maplibregl.Marker({color:'#ff465c'})
   .setLngLat([selectedDestination.lon,selectedDestination.lat])
   .setPopup(new maplibregl.Popup().setText(selectedDestination.name))
   .addTo(map);

  if(position)await buildRoute();
  else map.flyTo({center:[selectedDestination.lon,selectedDestination.lat],zoom:15});
 }catch(e){$('notice').innerText='Search Error: '+e.message}
}

// =====================================
// ROUTING
// =====================================
async function buildRoute(){
 if(!position||!selectedDestination)return;
 $('notice').innerText='Building route...';

 try{
  const a=position,b=selectedDestination;
  const url=`https://router.project-osrm.org/route/v1/driving/${a.lon},${a.lat};${b.lon},${b.lat}?overview=full&geometries=geojson&steps=true`;

  const response=await fetch(url);
  const data=await response.json();
  if(data.code!=='Ok')throw Error('Route not available');

  route=data.routes[0];
  routeSteps=route.legs[0].steps;
  routeCoords=route.geometry.coordinates;

  if(map.getLayer('route-line'))map.removeLayer('route-line');
  if(map.getSource('route'))map.removeSource('route');

  map.addSource('route',{type:'geojson',data:{
   type:'Feature',geometry:route.geometry,properties:{}
  }});
  map.addLayer({
   id:'route-line',type:'line',source:'route',
   layout:{'line-join':'round','line-cap':'round'},
   paint:{'line-color':'#08bfff','line-width':6}
  });

  const bounds=new maplibregl.LngLatBounds(routeCoords[0],routeCoords[0]);
  routeCoords.forEach(c=>bounds.extend(c));
  map.fitBounds(bounds,{padding:45});

  lastInstruction=-1;
  updateProgress();
  showSteps();

  $('notice').innerText='Navigation route ready';
  saveHistory();
 }catch(e){$('notice').innerText='Routing error: '+e.message}
}

function distance(a,b){
 const R=6371000;
 const rad=x=>x*Math.PI/180;
 const dLat=rad(b[1]-a[1]),dLon=rad(b[0]-a[0]);
 const x=Math.sin(dLat/2)**2+
 Math.cos(rad(a[1]))*Math.cos(rad(b[1]))*Math.sin(dLon/2)**2;
 return 2*R*Math.atan2(Math.sqrt(x),Math.sqrt(1-x));
}

function updateProgress(){
 if(!route||!position)return;
 const here=[position.lon,position.lat];
 let nearest=Infinity,index=0;

 routeCoords.forEach((c,i)=>{
  const d=distance(here,c);
  if(d<nearest){nearest=d;index=i}
 });

 let remaining=0;
 for(let i=index;i<routeCoords.length-1;i++){
  remaining+=distance(routeCoords[i],routeCoords[i+1]);
 }
 $('distance').innerText=(remaining/1000).toFixed(1);
 const speed=Math.max(20,Number($('speed').innerText)||20);
 $('eta').innerText=Math.ceil(remaining/(speed/3.6)/60)+' min';

 let next=-1,nextDistance=Infinity;
 routeSteps.forEach((s,i)=>{
  const d=distance(here,s.maneuver.location);
  if(d<nextDistance && d>15){nextDistance=d;next=i}
 });
 if(next>=0){
  const m=routeSteps[next].maneuver;
  const instruction=m.type+' '+(m.modifier||'');
  $('nextTurn').innerText='➡️ '+instruction+' | '+Math.round(nextDistance)+' m';
  if(next!==lastInstruction && nextDistance<350){
   speak(voiceLanguage==='bn-BD'?
    'সামনে '+Math.round(nextDistance)+' মিটার পরে '+instruction:
    'In '+Math.round(nextDistance)+' meters, '+instruction);
   lastInstruction=next;
  }
 }

 // Off-route detection and automatic recalculation
 if(nearest>100 && !rerouting && Date.now()-lastSave>15000){
  rerouting=true;
  $('notice').innerText='Route থেকে সরে গেছেন। নতুন পথ খোঁজা হচ্ছে...';
  lastSave=Date.now();
  buildRoute().finally(()=>rerouting=false);
 }
}

function showSteps(){
 $('steps').innerHTML=routeSteps.slice(0,15).map((s,i)=>
  `${i+1}. ${s.maneuver.type} ${s.maneuver.modifier||''} — ${Math.round(s.distance)}m`
 ).join('<br>');
}

// =====================================
// 3D BUILDINGS
// =====================================
function toggle3D(){
 threeD=!threeD;
 map.easeTo({pitch:threeD?60:0,bearing:threeD?15:0,duration:900});
 if(threeD)add3D();
}

function add3D(){
 const layers=map.getStyle().layers||[];
 const b=layers.find(x=>x['source-layer']==='building'&&x.type==='fill');
 if(!b)return;
 if(map.getLayer('mahir-3d'))return;

 const first=layers.find(x=>x.type==='symbol')?.id;
 map.addLayer({
  id:'mahir-3d',source:b.source,'source-layer':b['source-layer'],
  type:'fill-extrusion',minzoom:14,
  paint:{
   'fill-extrusion-color':'#9baec2',
   'fill-extrusion-height':['coalesce',['get','render_height'],['get','height'],10],
   'fill-extrusion-base':['coalesce',['get','render_min_height'],0],
   'fill-extrusion-opacity':.8
  }
 },first);
}

// =====================================
// SATELLITE
// =====================================
function toggleSatellite(){
 satellite=!satellite;
 if(satellite){
  if(!map.getSource('satellite')){
   map.addSource('satellite',{
    type:'raster',
    tiles:['https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}'],
    tileSize:256,
    attribution:'Esri'
   });
  }
  if(!map.getLayer('satellite-layer'))
   map.addLayer({id:'satellite-layer',type:'raster',source:'satellite'},'waterway-label');
 }else if(map.getLayer('satellite-layer')){
  map.removeLayer('satellite-layer');
 }
}

// =====================================
// ROAD CAMERAS & TRAFFIC LIGHTS
// =====================================
async function loadRoads(){
 const b=map.getBounds();
 $('notice').innerText='Loading road information...';
 const q=`[out:json][timeout:25];(
 node["highway"="speed_camera"](${b.getSouth()},${b.getWest()},${b.getNorth()},${b.getEast()});
 node["highway"="traffic_signals"](${b.getSouth()},${b.getWest()},${b.getNorth()},${b.getEast()});
 node["traffic_signals"="yes"](${b.getSouth()},${b.getWest()},${b.getNorth()},${b.getEast()});
 );out body;`;

 try{
  const r=await fetch('https://overpass-api.de/api/interpreter',{
   method:'POST',body:'data='+encodeURIComponent(q)
  });
  if(!r.ok)throw Error('Road server unavailable');
  const data=await r.json();

  roadMarkers.forEach(m=>m.remove());
  roadMarkers=[];cameraData=[];
  let signals=0,cameras=0;

  data.elements.forEach(n=>{
   const isCamera=n.tags.highway==='speed_camera';
   if(isCamera){cameras++;cameraData.push([n.lon,n.lat])}
   else signals++;

   const marker=new maplibregl.Marker({
    color:isCamera?'#ff4646':'#ffbd36'
   }).setLngLat([n.lon,n.lat])
   .setPopup(new maplibregl.Popup().setText(
    isCamera?'📸 Mapped speed camera':'🚦 Traffic signal (no live timer)'
   )).addTo(map);
   roadMarkers.push(marker);
  });
  $('notice').innerText=`Cameras: ${cameras} | Traffic signals: ${signals}`;
 }catch(e){$('notice').innerText='Road data unavailable'}
}

function checkCameras(){
 if(!position||!cameraData.length)return;
 let nearest=Infinity;
 cameraData.forEach(c=>nearest=Math.min(nearest,distance(
  [position.lon,position.lat],c
 )));
 if(nearest<500){
  $('alert').innerText='📸 Mapped camera nearby: '+Math.round(nearest)+'m';
  if(nearest<200 && Date.now()-lastAlert>15000){
   speak(voiceLanguage==='bn-BD'?'সামনে স্পিড ক্যামেরা':'Speed camera ahead');
   if(navigator.vibrate)navigator.vibrate(250);
   lastAlert=Date.now();
  }
 }
}

// =====================================
// NEARBY PLACES
// =====================================
async function loadPOI(type){
 const b=map.getBounds();
 const tag={
  fuel:'["amenity"="fuel"]',
  hospital:'["amenity"="hospital"]',
  parking:'["amenity"="parking"]',
  restaurant:'["amenity"="restaurant"]',
  place_of_worship:'["amenity"="place_of_worship"]',
  atm:'["amenity"="atm"]',
  charging_station:'["amenity"="charging_station"]'
 }[type];

 $('notice').innerText='Nearby places loading...';
 try{
  const q=`[out:json][timeout:25];(node${tag}(${b.getSouth()},${b.getWest()},${b.getNorth()},${b.getEast()});way${tag}(${b.getSouth()},${b.getWest()},${b.getNorth()},${b.getEast()}););out center tags;`;
  const r=await fetch('https://overpass-api.de/api/interpreter',{
   method:'POST',body:'data='+encodeURIComponent(q)
  });
  if(!r.ok)throw Error();
  const d=await r.json();
  poiMarkers.forEach(m=>m.remove());poiMarkers=[];

  d.elements.forEach(p=>{
   const lon=p.lon??p.center?.lon,lat=p.lat??p.center?.lat;
   if(!Number.isFinite(lat)||!Number.isFinite(lon))return;
   const m=new maplibregl.Marker({color:'#18d78b'})
    .setLngLat([lon,lat])
    .setPopup(new maplibregl.Popup().setText(p.tags?.name||type))
    .addTo(map);
   poiMarkers.push(m);
  });
  $('notice').innerText=d.elements.length+' nearby places found';
 }catch(e){$('notice').innerText='Nearby search unavailable'}
}

// =====================================
// CLOUD FAVORITES
// =====================================
async function savePlace(){
 if(!position)return alert('আগে GPS চালু করুন');
 const name=prompt('Place name লিখুন:');
 if(!name)return;
 try{
  const r=await backend('savePlace',{
   name,lat:position.lat,lon:position.lon
  });
  $('notice').innerText='☁️ Place saved to cloud';
 }catch(e){
  alert('Cloud save failed: '+e.message);
 }
}

async function showSaved(){
 try{
  const r=await backend('getPlaces');
  const list=r.places||[];
  $('savedList').innerHTML=list.map((p,i)=>
   `<div><button onclick="goSaved(${i})">${i+1}. ${escapeHTML(p.name)}</button></div>`
  ).join('');
  savedPlaces=list;
 }catch(e){alert(e.message)}
}

function goSaved(i){
 const p=savedPlaces[i];if(!p)return;
 selectedDestination={lat:p.lat,lon:p.lon,name:p.name};
 if(destinationMarker)destinationMarker.remove();
 destinationMarker=new maplibregl.Marker({color:'#ff4356'})
  .setLngLat([p.lon,p.lat}).addTo(map);
 map.flyTo({center:[p.lon,p.lat],zoom:15});
 if(position)buildRoute();
}

async function saveHistory(){
 if(!selectedDestination)return;
 try{
  await backend('saveHistory',{
   destination:selectedDestination.name,
   lat:selectedDestination.lat,
   lon:selectedDestination.lon
  });
 }catch(e){console.warn(e)}
}

async function loadHistory(){
 try{
  const r=await backend('getHistory');
  $('savedList').innerHTML=(r.history||[]).map((p,i)=>
   `<div>${i+1}. ${escapeHTML(p.destination)}</div>`
  ).join('');
 }catch(e){alert(e.message)}
}

function escapeHTML(s){
 return String(s).replace(/[&<>"']/g,c=>({
 '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'
 })[c]);
}

// =====================================
// SPEED LIMIT
// =====================================
function changeLimit(){
 const n=Number(prompt('Road speed limit KM/H:',speedLimit));
 if(n>0&&n<300){
  speedLimit=n;
  $('limit').innerText=n;
 }
}

// =====================================
// LOCATION SHARING
// =====================================
function shareLocation(){
 if(!position)return alert('GPS চালু করুন');
 const url=`https://www.google.com/maps?q=${position.lat},${position.lon}`;
 if(navigator.share){
  navigator.share({title:'My GPS Location',text:'My current location',url});
 }else if(navigator.clipboard){
  navigator.clipboard.writeText(url).then(()=>alert('Location link copied'));
 }else alert(url);
}

// =====================================
// START
// =====================================
initBackend();
$('limit').innerText=speedLimit;

</script>
</body>
</html>
