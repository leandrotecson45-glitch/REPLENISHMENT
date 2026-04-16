<!DOCTYPE html>
<html>
<head>
  <title>Fieldworker Portal - FINAL WITH START PHOTO</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <!-- Leaflet -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

  <!-- EXIF -->
  <script src="https://cdn.jsdelivr.net/npm/exif-js"></script>

  <!-- Excel -->
  <script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>

  <style>
    body{font-family:Arial;background:black;color:White;padding:10px;}
    header{text-align:center;background:linear-gradient(45deg,#22c55e,#0ea5e9);padding:10px;}
    input,select,button{padding:8px;margin:5px;border-radius:8px;border:none;}
    button{background:#22c55e;color:white;cursor:pointer;}
    #map{height:400px;width:95%;margin:10px auto;}
    table{width:95%;margin:10px auto;border-collapse:collapse;}
    th,td{border:1px solid #ccc;padding:6px;text-align:center;}
    img{width:60px;border-radius:6px;}
    .day-row{background:black;font-weight:bold;}
  </style>
</head>
<body>

<header>📸🚀 Fieldworker Portal - FINAL WITH START PHOTO</header>

<div style="text-align:center;">
  Name: <input type="text" id="name"><br>

  Vehicle:
  <select id="vehicle">
    <option value="motor">Motor (₱10.4/km)</option>
    <option value="car">4 Wheels (₱20.4/km)</option>
  </select><br>

  Upload Photos: <input type="file" id="upload" multiple accept="image/*"><br>

  <button onclick="exportJSON()">Download JSON</button>
  <button onclick="exportExcel()">Download Excel</button>
</div>

<div id="map"></div>

<table>
<thead>
<tr>
<th>Date</th><th>Photo</th><th>Route</th><th>From</th><th>To</th><th>KM</th><th>₱</th>
</tr>
</thead>
<tbody id="tableBody"></tbody>
</table>

<script>
const API_KEY = "eyJvcmciOiI1YjNjZTM1OTc4NTExMTAwMDFjZjYyNDgiLCJpZCI6ImEyOTE3YTc0Yzc2NzQ5NmJiOTg3ZDljNzMzMzY1OTFiIiwiaCI6Im11cm11cjY0In0=";

let trips=[];
let map=L.map('map').setView([14.6,121.0],10);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

let markers=[],routes=[];
let isUpdating=false;

// Upload photos
document.getElementById("upload").addEventListener("change", e=>{
  [...e.target.files].forEach(readPhoto);
});

// Read EXIF + image
function readPhoto(file){
  const reader=new FileReader();
  reader.onload=e=>{
    const img=e.target.result;

    EXIF.getData(file,function(){
      let lat=EXIF.getTag(this,"GPSLatitude");
      let lng=EXIF.getTag(this,"GPSLongitude");
      let latRef=EXIF.getTag(this,"GPSLatitudeRef");
      let lngRef=EXIF.getTag(this,"GPSLongitudeRef");

      if(!lat || !lng){
        alert("No GPS: "+file.name);
        return;
      }

      lat=convert(lat,latRef);
      lng=convert(lng,lngRef);

      let date=new Date().toISOString().split('T')[0];

      trips.push({lat,lng,date,file:file.name,image:img});
      update();
    });
  };
  reader.readAsDataURL(file);
}

function convert(dms,ref){
  let dd=dms[0]+dms[1]/60+dms[2]/3600;
  if(ref=="S"||ref=="W") dd*=-1;
  return dd;
}

// MAIN UPDATE FUNCTION
async function update(){
  if(isUpdating) return;
  isUpdating=true;

  markers.forEach(m=>map.removeLayer(m));
  routes.forEach(r=>map.removeLayer(r));
  markers=[]; 
  routes=[];

  let grouped={};
  trips.forEach(t=>{
    if(!grouped[t.date]) grouped[t.date]=[];
    grouped[t.date].push(t);
  });

  let tb=document.getElementById("tableBody");
  tb.innerHTML="";

  for(let date in grouped){
    let arr=grouped[date];
    let vehicle=document.getElementById("vehicle").value;
    let rate=(vehicle==="motor")?10.4:20.4;

    let totalKM=0,totalMoney=0;

    tb.innerHTML+=`<tr class="day-row">
      <td colspan="7">📅 ${date}</td>
    </tr>`;

    // SHOW FIRST POINT
    tb.innerHTML+=`
    <tr>
      <td>${date}</td>
      <td><img src="${arr[0].image}"></td>
      <td>Start</td>
      <td>-</td>
      <td>${arr[0].lat.toFixed(6)}, ${arr[0].lng.toFixed(6)}</td>
      <td>0</td>
      <td>0</td>
    </tr>`;

    // DRAW MARKERS
    arr.forEach((t,i)=>{
      let m=L.marker([t.lat,t.lng]).addTo(map);
      m.bindPopup(`<b>Point ${i+1}</b><br>${t.file}<br><img src="${t.image}" width="120">`);
      markers.push(m);
    });

    // DRAW SEGMENTS
    for(let i=1;i<arr.length;i++){
      let A=arr[i-1];
      let B=arr[i];

      let km=0;

      try{
        let res=await fetch("https://api.openrouteservice.org/v2/directions/driving-car/geojson",{
          method:"POST",
          headers:{
            "Authorization":API_KEY,
            "Content-Type":"application/json"
          },
          body:JSON.stringify({coordinates:[[A.lng,A.lat],[B.lng,B.lat]]})
        });

        let data=await res.json();

        if(data.features){
          let route=L.geoJSON(data,{style:{color:"white",weight:5}});
          route.addTo(map);
          routes.push(route);

          km=data.features[0].properties.summary.distance/1000;
        }else{
          km=distance(A,B);
          drawFallback(A,B);
        }

      }catch(e){
        km=distance(A,B);
        drawFallback(A,B);
      }

      let money=km*rate;
      totalKM+=km;
      totalMoney+=money;

      tb.innerHTML+=`
      <tr>
        <td>${date}</td>
        <td><img src="${B.image}"></td>
        <td>${i} ➝ ${i+1}</td>
        <td>${A.lat.toFixed(6)}, ${A.lng.toFixed(6)}</td>
        <td>${B.lat.toFixed(6)}, ${B.lng.toFixed(6)}</td>
        <td>${km.toFixed(2)}</td>
        <td>${money.toFixed(2)}</td>
      </tr>`;
    }

    tb.innerHTML+=`
    <tr class="day-row">
      <td colspan="5">TOTAL</td>
      <td>${totalKM.toFixed(2)} KM</td>
      <td>₱ ${totalMoney.toFixed(2)}</td>
    </tr>`;
  }

  if(trips.length>0){
    map.setView([trips[0].lat,trips[0].lng],12);
  }

  isUpdating=false;
}

// fallback line
function drawFallback(A,B){
  let line=L.polyline([[A.lat,A.lng],[B.lat,B.lng]],{color:"yellow",weight:4}).addTo(map);
  routes.push(line);
}

// haversine
function distance(a,b){
  const R=6371;
  const dLat=(b.lat-a.lat)*Math.PI/180;
  const dLng=(b.lng-a.lng)*Math.PI/180;
  const lat1=a.lat*Math.PI/180;
  const lat2=b.lat*Math.PI/180;
  const x=Math.sin(dLat/2)**2 + Math.cos(lat1)*Math.cos(lat2)*Math.sin(dLng/2)**2;
  return R*2*Math.atan2(Math.sqrt(x),Math.sqrt(1-x));
}

// JSON
function exportJSON(){
  let blob=new Blob([JSON.stringify(trips,null,2)],{type:"application/json"});
  let a=document.createElement("a");
  a.href=URL.createObjectURL(blob);
  a.download="fieldworker.json";
  a.click();
}

// Excel
function exportExcel(){
  let wb=XLSX.utils.book_new();

  let grouped={};
  trips.forEach(t=>{
    if(!grouped[t.date]) grouped[t.date]=[];
    grouped[t.date].push(t);
  });

  for(let date in grouped){
    let arr=grouped[date];
    let data=[];

    // ADD FIRST POINT
    data.push({
      Route:"Start",
      From:"-",
      To:`${arr[0].lat},${arr[0].lng}`,
      KM:0,
      Allowance:0,
      Photo:arr[0].file
    });

    for(let i=1;i<arr.length;i++){
      let A=arr[i-1];
      let B=arr[i];
      let km=distance(A,B);
      let rate=(document.getElementById("vehicle").value==="motor")?10.4:20.4;

      data.push({
        Route:`${i} -> ${i+1}`,
        From:`${A.lat},${A.lng}`,
        To:`${B.lat},${B.lng}`,
        KM:km.toFixed(2),
        Allowance:(km*rate).toFixed(2),
        Photo:B.file
      });
    }

    let ws=XLSX.utils.json_to_sheet(data);
    XLSX.utils.book_append_sheet(wb,ws,date);
  }

  XLSX.writeFile(wb,"Fieldworker_Report.xlsx");
}
</script>

</body>
</html>
