# piano-detector


<!DOCTYPE html>

<html lang="es">

<head>

<meta charset="UTF-8">

<meta name="viewport" content="width=device-width, initial-scale=1">

<title>Piano Detector</title>

<style>

*{box-sizing:border-box}

body{

 margin:0;padding:20px;

 background:#101522;color:#f4f7fb;

 font-family:system-ui,sans-serif;

}

main{max-width:900px;margin:auto}

h1{color:#75e0c1}

.panel{

 background:#1a2436;padding:18px;

 border:1px solid #34435c;

 border-radius:16px;margin:18px 0;

}

button{

 padding:12px 16px;border:0;

 border-radius:10px;margin:5px;

 font-weight:bold;background:#75e0c1;

 color:#10201d;cursor:pointer;

}

button:disabled{opacity:.4}

#status{color:#75e0c1;margin-top:12px}

#detected{font-size:23px;color:#75e0c1}

.piano{

 display:flex;position:relative;

 height:140px;max-width:700px;margin:20px auto;

}

.white{

 flex:1;background:#f4f1e9;

 border:1px solid #aaa;position:relative;

 border-radius:0 0 5px 5px;

}

.white span{

 position:absolute;bottom:5px;

 color:#222;font-size:10px;width:100%;

 text-align:center;

}

.black{

 position:absolute;top:0;width:7%;

 height:85px;background:#151923;

 border-radius:0 0 5px 5px;

 z-index:2;transform:translateX(-50%);

}

.white.active{background:#75e0c1}

.black.active{background:#e8b85c}

canvas{width:100%;background:#f8f7f2;border-radius:10px}

.notes{display:flex;flex-wrap:wrap;gap:8px}

.chip{background:#293951;padding:8px;border-radius:20px}

</style>

</head>

<body>

<main>

<h1>🎹 Piano Detector</h1>

<p>Un piano que escucha y escribe las notas que detecta.</p>

<section class="panel">

<button id="start">🎙️ Activar micrófono</button>

<button id="stop" disabled>Detener</button>

<button id="clear">Borrar pentagrama</button>

<p id="status">Micrófono apagado.</p>

<div id="detected">Esperando sonido...</div>

</section>

<section class="panel">

<h2>Teclado detectado</h2>

<div class="piano" id="piano"></div>

</section>

<section class="panel">

<h2>Pentagrama</h2>

<canvas id="staff" width="900" height="250"></canvas>

<div class="notes" id="noteList"></div>

</section>

</main>

<script>

const $=id=>document.getElementById(id);

const names=['Do','Do♯','Re','Re♯','Mi','Fa','Fa♯','Sol','Sol♯','La','La♯','Si'];

let ctx=null,analyser=null,stream=null,raf=0;

let history=[],lastUpdate=0;

const whiteMidis=[48,50,52,53,55,57,59,60,62,64,65,67,69,71,72];

whiteMidis.forEach(m=>{

 const k=document.createElement('div');

 k.className='white';k.dataset.midi=m;

 k.innerHTML='<span>'+names[m%12]+'</span>';

 $('piano').append(k);

});

[1,2,4,5,6,8,9,11,12,13].forEach((p,i)=>{

 const k=document.createElement('div');

 k.className='black';

 k.dataset.midi=48+p;

 k.style.left=((i+1)/15*100)+'%';

 $('piano').append(k);

});

function midiName(m){

 return names[m%12]+' '+(Math.floor(m/12)-1);

}

function freqMidi(f){

 return Math.round(69+12*Math.log2(f/440));

}

function detect(){

 const data=new Uint8Array(analyser.frequencyBinCount);

 analyser.getByteFrequencyData(data);

 const peaks=[];

 const sr=ctx.sampleRate;

 const size=analyser.fftSize;

 for(let i=2;i<data.length-2;i++){

  const f=i*sr/size;

  if(f<75||f>1800)continue;

  if(data[i]>data[i-1]&&data[i]>=data[i+1]&&data[i]>13){

   const m=freqMidi(f);

   const target=440*Math.pow(2,(m-69)/12);

   const cents=Math.abs(1200*Math.log2(f/target));

   if(cents<48)peaks.push({m,amp:data[i]});

  }

 }

 peaks.sort((a,b)=>b.amp-a.amp);

 const result=[];

 for(const p of peaks){

  if(!result.includes(p.m))result.push(p.m);

  if(result.length===6)break;

 }

 return result.sort((a,b)=>a-b);

}

function updateKeys(ms){

 document.querySelectorAll('.white,.black').forEach(k=>{

  k.classList.toggle('active',ms.includes(+k.dataset.midi));

 });

}

function drawStaff(){

 const c=$('staff'),g=c.getContext('2d');

 g.clearRect(0,0,c.width,c.height);

 g.fillStyle='#f8f7f2';

 g.fillRect(0,0,c.width,c.height);

 g.strokeStyle='#333';g.lineWidth=1.5;

 [58,168].forEach(y=>{

  for(let i=0;i<5;i++){

   g.beginPath();

   g.moveTo(35,y+i*10);

   g.lineTo(870,y+i*10);

   g.stroke();

  }

 });

 g.fillStyle='#222';

 g.font='bold 48px serif';

 g.fillText('𝄞',42,105);

 g.fillText('𝄢',42,215);

 g.font='13px sans-serif';

 g.fillText('Clave de sol',105,35);

 g.fillText('Clave de fa',105,145);

 history.slice(-38).forEach((m,i)=>{

  const x=145+i*19%700;

  const y=m>=60?98-(m-64)*5:208-(m-43)*5;

  g.fillStyle='#172235';

  g.beginPath();

  g.ellipse(x,y,7,4.5,-.15,0,Math.PI*2);

  g.fill();

  g.strokeStyle='#172235';

  g.beginPath();

  g.moveTo(x+6,y);

  g.lineTo(x+6,y-28);

  g.stroke();

  g.font='10px sans-serif';

  g.fillText(midiName(m),x-12,235);

 });

}

function loop(t){

 if(!analyser)return;

 if(t-lastUpdate>110){

  lastUpdate=t;

  const ms=detect();

  if(ms.length){

   $('detected').textContent=ms.map(midiName).join(' + ');

   updateKeys(ms);

   ms.forEach(m=>{

    if(!history.includes(m))history.push(m);

   });

   drawStaff();

   $('noteList').innerHTML=ms.map(m=>

    '<span class="chip">'+midiName(m)+'</span>'

   ).join('');

  }else{

   $('detected').textContent='Escuchando...';

   updateKeys([]);

   $('noteList').innerHTML='';

  }

 }

 raf=requestAnimationFrame(loop);

}

$('start').onclick=async()=>{

 try{

  stream=await navigator.mediaDevices.getUserMedia({

   audio:{

    echoCancellation:false,

    noiseSuppression:false,

    autoGainControl:false

   }

  });

  ctx=new (window.AudioContext||window.webkitAudioContext)();

  await ctx.resume();

  analyser=ctx.createAnalyser();

  analyser.fftSize=8192;

  analyser.smoothingTimeConstant=.72;

  const source=ctx.createMediaStreamSource(stream);

  source.connect(analyser);

  $('status').textContent='🎙️ Micrófono activo. Escuchando piano.';

  $('start').disabled=true;

  $('stop').disabled=false;

  loop(0);

 }catch(e){

  $('status').textContent='Error de micrófono: '+e.message;

 }

};

$('stop').onclick=()=>{

 cancelAnimationFrame(raf);

 if(stream)stream.getTracks().forEach(t=>t.stop());

 if(ctx)ctx.close();

 stream=null;ctx=null;analyser=null;

 $('start').disabled=false;

 $('stop').disabled=true;

 $('status').textContent='Micrófono apagado.';

 $('detected').textContent='Escucha detenida';

 updateKeys([]);

};

$('clear').onclick=()=>{

 history=[];

 drawStaff();

 $('noteList').innerHTML='';

};

drawStaff();

</script>

</body>

</html>