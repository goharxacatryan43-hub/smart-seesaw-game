(() => {
  const $ = id => document.getElementById(id);
  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x9fd0ed);
  scene.fog = new THREE.Fog(0x9fd0ed, 24, 65);
  const camera = new THREE.PerspectiveCamera(55, innerWidth/innerHeight, .1, 150);
  camera.position.set(9, 8, 12);
  const renderer = new THREE.WebGLRenderer({antialias:true});
  renderer.setSize(innerWidth,innerHeight);
  renderer.setPixelRatio(Math.min(devicePixelRatio,2));
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  $('game').appendChild(renderer.domElement);

  scene.add(new THREE.HemisphereLight(0xffffff,0x66805b,2.0));
  const sun = new THREE.DirectionalLight(0xffffff,2.2);
  sun.position.set(-8,15,10); sun.castShadow=true; sun.shadow.mapSize.set(2048,2048); scene.add(sun);

  const mat = (color, roughness=.8) => new THREE.MeshStandardMaterial({color,roughness});
  const grass=mat(0x77b96a), wood=mat(0xb9783f), darkWood=mat(0x75472b), metal=mat(0x526c7b,.35);
  const ground=new THREE.Mesh(new THREE.PlaneGeometry(100,100),grass);ground.rotation.x=-Math.PI/2;ground.position.y=-.08;ground.receiveShadow=true;scene.add(ground);
  const grid=new THREE.GridHelper(80,40,0x659b5c,0x83b77a);grid.position.y=-.065;scene.add(grid);

  // Decorative playground trees and clouds
  function tree(x,z){const trunk=new THREE.Mesh(new THREE.CylinderGeometry(.16,.22,1.7,8),darkWood);trunk.position.set(x,.8,z);trunk.castShadow=true;scene.add(trunk);const crown=new THREE.Mesh(new THREE.SphereGeometry(.85,12,10),mat(0x398b55));crown.position.set(x,2,z);crown.castShadow=true;scene.add(crown)}
  [[-8,-6],[-10,3],[8,-7],[10,4],[-5,9],[6,9]].forEach(p=>tree(...p));

  // Seesaw support and beam
  const support=new THREE.Mesh(new THREE.CylinderGeometry(.18,.3,1.5,5),mat(0x647d8d));
  support.position.set(0,.72,0);support.castShadow=true;scene.add(support);
  const pivot=new THREE.Mesh(new THREE.CylinderGeometry(.22,.22,.25,24),metal);
  pivot.rotation.z=Math.PI/2;pivot.position.y=1.48;scene.add(pivot);
  const beamGroup=new THREE.Group();beamGroup.position.set(0,1.55,0);scene.add(beamGroup);
  const beam=new THREE.Mesh(new THREE.BoxGeometry(7.2,.22,.48),wood);beam.castShadow=true;beam.receiveShadow=true;beamGroup.add(beam);
  const stripe=new THREE.Mesh(new THREE.BoxGeometry(7.15,.035,.49),mat(0xe5b56b));stripe.position.y=.13;beamGroup.add(stripe);
  function seat(x){const s=new THREE.Mesh(new THREE.BoxGeometry(.95,.16,.75),mat(0xe4a343));s.position.set(x,.25,0);s.castShadow=true;beamGroup.add(s);const back=new THREE.Mesh(new THREE.BoxGeometry(.12,.48,.72),mat(0xc77a32));back.position.set(x+Math.sign(x)*.42,.48,0);beamGroup.add(back);const foot=new THREE.Mesh(new THREE.BoxGeometry(.55,.08,.12),metal);foot.position.set(x,-.04,0);beamGroup.add(foot)}
  seat(-2.65);seat(2.65);
  const endCaps=[];
  [-3.5,3.5].forEach(x=>{const c=new THREE.Mesh(new THREE.SphereGeometry(.14,12,10),mat(0x315f82));c.position.set(x,0,0);beamGroup.add(c);endCaps.push(c)});

  // Character builder: recognizable stylized people, different sizes and weights
  const palettes={adult:0x2879c5,child:0xe5a23b,toddler:0xe66c72,robot:0x8496a6};
  const defaults={adult:60,child:30,toddler:18,robot:45};
  function makeCharacter(type,color=palettes[type]||0x2879c5){
    const g=new THREE.Group();const scale=type==='adult'?1.08:type==='toddler'?.65:type==='robot'?.95:.82;
    const bodyMat=mat(color), skin=mat(type==='robot'?0xb8c6d0:0xf0c49a), pants=mat(0x344b68), white=mat(0xffffff), black=mat(0x243342);
    const torso=new THREE.Mesh(new THREE.CapsuleGeometry(.28,.55,5,10),bodyMat);torso.position.y=1.05;torso.castShadow=true;g.add(torso);
    const head=new THREE.Mesh(new THREE.SphereGeometry(.25,16,12),skin);head.position.y=1.62;head.castShadow=true;g.add(head);
    const eyeGeo=new THREE.SphereGeometry(.035,8,8);
    [-.09,.09].forEach(x=>{const e=new THREE.Mesh(eyeGeo,black);e.position.set(x,1.66,.22);g.add(e)});
    for(const x of [-.14,.14]){const leg=new THREE.Mesh(new THREE.CapsuleGeometry(.09,.42,4,8),pants);leg.position.set(x,.36,0);g.add(leg);const arm=new THREE.Mesh(new THREE.CapsuleGeometry(.07,.38,4,8),skin);arm.position.set(x*2,1.08,0);arm.rotation.z=x<0?-.3:.3;g.add(arm)}
    g.scale.setScalar(scale);g.userData={type,scale};return g;
  }
  const myId='p'+Math.random().toString(36).slice(2,8);
  const me={id:myId,name:'Դու',type:'adult',weight:60,side:1,distance:2,seated:false,x:0,z:3,rotation:0};
  const myChar=makeCharacter(me.type);scene.add(myChar);
  const remote={};
  const colors=[0x2879c5,0xe5a23b,0xe66c72,0x4f9c83,0x8d6ac2];
  function addRemote(id,data){if(remote[id])return;const obj=makeCharacter(data.type||'child',data.color||colors[Object.keys(remote).length%colors.length]);scene.add(obj);remote[id]={obj,data:{...data}};updatePlayers()}
  function removeRemote(id){if(remote[id]){scene.remove(remote[id].obj);delete remote[id];updatePlayers()}}
  function setCharacter(type){me.type=type;me.weight=defaults[type];$('weight').value=me.weight;$('weightValue').textContent=me.weight;scene.remove(myChar);const fresh=makeCharacter(type);scene.add(fresh);playerObject=fresh;toast('Կերպարը փոխվեց։')}
  let playerObject=myChar;

  // Peer-to-peer room: host creates a room code; guests connect to host's PeerJS ID.
  let peer=null, hostConn=null, isHost=false, roomCode='';
  const peers={};
  function setStatus(t){$('connection').textContent=t}
  function toast(t){$('toast').textContent=t;$('toast').classList.add('show');clearTimeout(toast.timer);toast.timer=setTimeout(()=>$('toast').classList.remove('show'),2400)}
  function updatePlayers(){
    const list=$('players');list.innerHTML='';
    const entries=[{id:myId,name:'Դու',type:me.type},...Object.entries(remote).map(([id,r])=>({id,name:r.data.name||'Ընկեր',type:r.data.type||'կերպար'}))];
    entries.forEach(p=>{const d=document.createElement('div');d.className='player-line';d.textContent=(p.id===myId?'🟢 ':'🔵 ')+p.name+' · '+p.type;list.appendChild(d)});
  }
  function statePacket(){return {id:myId,name:'Խաղացող',type:me.type,weight:me.weight,side:me.side,distance:me.distance,seated:me.seated,x:me.x,z:me.z,rotation:me.rotation,color:palettes[me.type]}}
  function sendAll(packet){Object.values(peers).forEach(c=>{if(c.open)c.send(packet)})}
  function handlePacket(conn,p){
    if(!p||typeof p!=='object')return;
    if(p.kind==='hello'){addRemote(p.state.id,p.state);conn.send({kind:'welcome',state:statePacket()});if(isHost)Object.values(peers).forEach(c=>{if(c!==conn&&c.open)c.send({kind:'snapshot',state:statePacket()})});}
    if(p.kind==='state'){if(!remote[p.state.id])addRemote(p.state.id,p.state);remote[p.state.id].data=p.state;placeRemote(p.state.id)}
    if(p.kind==='welcome'||p.kind==='snapshot'){if(p.state.id!==myId){addRemote(p.state.id,p.state);remote[p.state.id].data=p.state;placeRemote(p.state.id)}}
    if(p.kind==='bye')removeRemote(p.id);
  }
  function attachConnection(conn){
    peers[conn.peer]=conn;
    conn.on('open',()=>{conn.send({kind:'hello',state:statePacket()});setStatus('Առցանց · '+(Object.keys(peers).length+1)+' խաղացող');toast('Ընկերդ միացավ։')});
    conn.on('data',p=>handlePacket(conn,p));
    conn.on('close',()=>{removeRemote(conn.peer);delete peers[conn.peer];setStatus(isHost?'Սենյակը բաց է':'Միացված չէ')});
    conn.on('error',()=>toast('Կապի սխալ․ փորձիր նորից։'));
  }
  function createRoom(){
    if(!window.Peer){toast('Առցանց գրադարանը չբեռնվեց։ Ստուգիր ինտերնետը։');return}
    roomCode=$('roomInput').value.trim().toUpperCase()||('SWING'+Math.floor(100+Math.random()*900));
    $('roomInput').value=roomCode;isHost=true;
    peer=new Peer('smart-seesaw-'+roomCode);
    setStatus('Սենյակը բացվում է…');
    peer.on('open',id=>{$('roomInfo').textContent='Սենյակի կոդը՝ '+roomCode+'. Ուղարկիր այս կոդը ընկերներիդ։';setStatus('Դու սենյակի հյուրընկալողն ես');toast('Սենյակը ստեղծվեց։')});
    peer.on('connection',attachConnection);
    peer.on('error',err=>{if(err.type==='unavailable-id')toast('Այս կոդն արդեն զբաղված է։ Փորձիր ուրիշ կոդ։');else toast('Չհաջողվեց ստեղծել սենյակ։')});
  }
  function joinRoom(){
    const code=$('roomInput').value.trim().toUpperCase();if(!code){toast('Մուտքագրիր սենյակի կոդը։');return}
    if(!window.Peer){toast('Առցանց գրադարանը չբեռնվեց։');return}
    isHost=false;peer=new Peer();setStatus('Միանում է…');
    peer.on('open',()=>{hostConn=peer.connect('smart-seesaw-'+code,{reliable:true});attachConnection(hostConn);$('roomInfo').textContent='Միանում ես '+code+' սենյակին։'});
    peer.on('error',()=>{setStatus('Միացված չէ');toast('Սենյակը չգտնվեց։ Ստուգիր կոդը։')});
  }
  function placeRemote(id){const r=remote[id];if(!r)return;const d=r.data;const angle=beamGroup.rotation.z;const x=d.seated?(d.side||1)*(d.distance||2):d.x;const y=d.seated?1.9:0;const z=d.seated?-.1:(d.z||3);r.obj.position.set(x,y,z);r.obj.rotation.y=d.seated?0:d.rotation||0}
  $('createRoom').onclick=createRoom;$('joinRoom').onclick=joinRoom;
  window.addEventListener('beforeunload',()=>{sendAll({kind:'bye',id:myId});if(peer)peer.destroy()});

  // Controls
  const keys={};let jump=0,camAngle=0,dragging=false,lastX=0;
  addEventListener('keydown',e=>{keys[e.key.toLowerCase()]=true;if(e.code==='Space'){e.preventDefault();if(!me.seated&&jump<=0)jump=.22}if(e.key.toLowerCase()==='e')toggleSeat()});
  addEventListener('keyup',e=>keys[e.key.toLowerCase()]=false);
  renderer.domElement.addEventListener('pointerdown',e=>{dragging=true;lastX=e.clientX;renderer.domElement.setPointerCapture(e.pointerId)});
  renderer.domElement.addEventListener('pointerup',()=>dragging=false);
  renderer.domElement.addEventListener('pointermove',e=>{if(dragging){camAngle+=(e.clientX-lastX)*.005;lastX=e.clientX}});
  function toggleSeat(){me.seated=!me.seated;if(me.seated){me.side=me.x<0?-1:1;me.distance=Math.max(.5,Math.min(3,Math.abs(me.x)||2));me.x=me.side*me.distance;toast('Դու նստեցիր ճոճանակին։')}else{me.x=me.side*4.2;me.z=2.5;toast('Դու վեր կացար։')}}
  $('sitBtn').onclick=toggleSeat;
  $('character').onchange=e=>setCharacter(e.target.value);
  $('weight').oninput=e=>{me.weight=+e.target.value;$('weightValue').textContent=me.weight};
  $('seatPosition').oninput=e=>{me.distance=+e.target.value;$('distanceValue').textContent=me.distance.toFixed(1);if(me.seated)me.x=me.side*me.distance};
  $('resetBtn').onclick=()=>{me.seated=false;me.x=0;me.z=3;me.weight=defaults[me.type];$('weight').value=me.weight;$('weightValue').textContent=me.weight;$('weight').dispatchEvent(new Event('input'));toast('Փորձը վերակայվեց։')};
  let soundOn=true;$('soundBtn').onclick=()=>{soundOn=!soundOn;$('soundBtn').textContent=soundOn?'🔊':'🔇'};

  // A simplified rigid seesaw model: torque difference drives angular acceleration.
  let angularVelocity=0;
  function physics(dt){
    let left=0,right=0;
    if(me.seated){const m=me.weight*9.81*me.distance;if(me.side<0)left+=m;else right+=m}
    Object.values(remote).forEach(r=>{const d=r.data;if(d.seated){const m=(d.weight||30)*9.81*(d.distance||2);if((d.side||1)<0)left+=m;else right+=m}});
    const diff=right-left;
    const inertia=95+left*.012+right*.012;
    const accel=diff/inertia*.22 - angularVelocity*1.5 - beamGroup.rotation.z*2.2;
    angularVelocity+=accel*dt;
    angularVelocity=Math.max(-1.4,Math.min(1.4,angularVelocity));
    beamGroup.rotation.z+=angularVelocity*dt;
    beamGroup.rotation.z=Math.max(-.55,Math.min(.55,beamGroup.rotation.z));
    $('angle').textContent=(beamGroup.rotation.z*180/Math.PI).toFixed(1)+'°';
    $('leftMoment').textContent=Math.round(left)+' Ն·մ';$('rightMoment').textContent=Math.round(right)+' Ն·մ';
    $('balance').textContent=Math.abs(diff)<45?'Հավասարակշռված է':(diff>0?'Աջ կողմը ծանր է':'Ձախ կողմը ծանր է');
    $('balance').style.color=Math.abs(diff)<45?'#23834b':'#a45d22';
  }

  let last=performance.now(),sendTimer=0;
  function animate(now){
    requestAnimationFrame(animate);const dt=Math.min((now-last)/1000,.035);last=now;
    if(!me.seated){
      const speed=4*dt;
      if(keys.w||keys.arrowup)me.z-=speed;if(keys.s||keys.arrowdown)me.z+=speed;
      if(keys.a||keys.arrowleft)me.x-=speed;if(keys.d||keys.arrowright)me.x+=speed;
      me.x=Math.max(-7,Math.min(7,me.x));me.z=Math.max(-5,Math.min(7,me.z));
      if(jump>0)jump-=dt;
      playerObject.position.set(me.x,Math.max(0,jump*2.8),me.z);
      if(Math.abs(me.x)+Math.abs(me.z)>.1)playerObject.rotation.y=Math.atan2(me.x-me.x,1);
    }else{
      const x=me.side*me.distance;
      playerObject.position.set(x,1.9+Math.sin(beamGroup.rotation.z)*x, -.1);
      playerObject.rotation.y=me.side<0?Math.PI:0;
    }
    physics(dt);
    Object.keys(remote).forEach(placeRemote);
    const target=new THREE.Vector3(0,1.1,0);
    const radius=17;camera.position.set(Math.sin(camAngle)*radius,8,Math.cos(camAngle)*radius);camera.lookAt(target);
    sendTimer+=dt;if(sendTimer>.08){sendTimer=0;sendAll({kind:'state',state:statePacket()})}
    renderer.render(scene,camera);
  }
  updatePlayers();requestAnimationFrame(animate);
  addEventListener('resize',()=>{camera.aspect=innerWidth/innerHeight;camera.updateProjectionMatrix();renderer.setSize(innerWidth,innerHeight)});
})();
