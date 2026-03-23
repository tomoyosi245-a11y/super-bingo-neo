#   
  
<!DOCTYPE html>  
<html lang="ja">  
<head>  
<meta charset="UTF-8">  
<title>Event Bingo Ultimate</title>  
  
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js"></script>  
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-database-compat.js"></script>  
  
<style>  
body{  
font-family:sans-serif;  
text-align:center;  
background:#0f172a;  
color:white;  
}  
  
#board{  
display:grid;  
grid-template-columns:repeat(5,70px);  
gap:6px;  
justify-content:center;  
margin-top:20px;  
}  
  
.cell{  
background:white;  
color:black;  
border-radius:12px;  
height:70px;  
display:flex;  
align-items:center;  
justify-content:center;  
font-size:20px;  
font-weight:bold;  
}  
  
.hit{  
background:#facc15;  
}  
  
button{  
padding:12px 20px;  
margin:6px;  
border:none;  
border-radius:10px;  
background:#22c55e;  
color:white;  
font-size:16px;  
}  
  
#draw{  
font-size:70px;  
margin-top:10px;  
}  
  
#qr{  
margin-top:15px;  
}  
</style>  
</head>  
<body>  
  
<h1>🎉 EVENT BINGO 🎉</h1>  
  
<button onclick="createRoom()">ルーム作成</button>  
<button onclick="draw()">抽選</button>  
<button onclick="autoDraw()">自動抽選</button>  
  
<div id="room"></div>  
<div id="players"></div>  
<div id="draw"></div>  
<div id="history"></div>  
<div id="qr"></div>  
  
<div id="board"></div>  
  
<h2>ランキング</h2>  
<div id="ranking"></div>  
  
<audio id="drawSound" src="https://actions.google.com/sounds/v1/cartoon/clang_and_wobble.ogg"></audio>  
<audio id="bingoSound" src="https://actions.google.com/sounds/v1/crowd/large_crowd_cheer.ogg"></audio>  
  
<script>  
  
const firebaseConfig = {  
apiKey:"AIzaSyBaRpc7J7dwgvLLgzV5dM5YHDgIYa9bQWU",  
authDomain:"bibibingo-72f63.firebaseapp.com",  
databaseURL:"https://bibibingo-72f63-default-rtdb.firebaseio.com",  
projectId:"bibibingo-72f63"  
};  
  
firebase.initializeApp(firebaseConfig);  
const db = firebase.database();  
  
let params = new URLSearchParams(location.search);  
let room = params.get("room");  
let host = params.get("host");  
let screen = params.get("screen");  
  
let playerId = localStorage.getItem("bingoId");  
if(!playerId){  
playerId = Math.random().toString(36).substring(2);  
localStorage.setItem("bingoId",playerId);  
}  
  
let name = localStorage.getItem("bingoName");  
if(!name){  
name = prompt("名前入力");  
localStorage.setItem("bingoName",name);  
}  
  
let history = [];  
let autoMode = false;  
  
function createRoom(){  
const id = Math.random().toString(36).substring(2,7);  
location.href = "?room="+id+"&host=1";  
}  
  
if(room){  
  
document.getElementById("room").innerText="ROOM: "+room;  
  
const url = location.origin + location.pathname + "?room="+room;  
document.getElementById("qr").innerHTML =  
`<img src="https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=${url}">`;  
  
generateCard();  
  
db.ref("rooms/"+room+"/players/"+playerId).set({  
name:name,  
bingo:false  
});  
  
db.ref("rooms/"+room+"/players").on("value",snap=>{  
const players = snap.val()||{};  
document.getElementById("players").innerText =  
"参加人数: "+Object.keys(players).length;  
updateRanking(players);  
});  
  
db.ref("rooms/"+room+"/numbers").on("value",snap=>{  
const nums = snap.val()||[];  
history = nums;  
updateHits(nums);  
updateHistory();  
});  
  
}  
  
function generateCard(){  
const board = document.getElementById("board");  
board.innerHTML="";  
  
let nums = [];  
while(nums.length<25){  
let n = Math.floor(Math.random()*75)+1;  
if(!nums.includes(n)) nums.push(n);  
}  
  
nums.forEach(n=>{  
let c = document.createElement("div");  
c.className="cell";  
c.innerText=n;  
board.appendChild(c);  
});  
}  
  
function draw(){  
  
let num = Math.floor(Math.random()*75)+1;  
slotAnimation(num);  
  
document.getElementById("drawSound").play();  
  
db.ref("rooms/"+room+"/numbers").once("value")  
.then(snap=>{  
let list = snap.val()||[];  
  
if(list.includes(num)) return draw();  
  
list.push(num);  
db.ref("rooms/"+room+"/numbers").set(list);  
});  
}  
  
function slotAnimation(finalNumber){  
let count = 0;  
let slot = setInterval(()=>{  
document.getElementById("draw").innerText =  
Math.floor(Math.random()*75)+1;  
count++;  
if(count>15){  
clearInterval(slot);  
document.getElementById("draw").innerText = finalNumber;  
}  
},80);  
}  
  
function autoDraw(){  
autoMode = !autoMode;  
if(autoMode) loopDraw();  
}  
  
function loopDraw(){  
if(!autoMode) return;  
draw();  
setTimeout(loopDraw,4000);  
}  
  
function updateHits(list){  
document.querySelectorAll(".cell").forEach(c=>{  
if(list.includes(Number(c.innerText))){  
c.classList.add("hit");  
}  
});  
checkBingo();  
}  
  
function updateHistory(){  
document.getElementById("history").innerText =  
"履歴: "+history.join(", ");  
}  
  
function checkBingo(){  
  
const cells=[...document.querySelectorAll(".cell")];  
let grid=[];  
  
for(let i=0;i<5;i++){  
grid[i]=cells.slice(i*5,(i+1)*5)  
.map(c=>c.classList.contains("hit"));  
}  
  
let bingo=false;  
  
for(let i=0;i<5;i++){  
if(grid[i].every(v=>v)) bingo=true;  
if(grid.map(r=>r[i]).every(v=>v)) bingo=true;  
}  
  
if(grid.map((r,i)=>r[i]).every(v=>v)) bingo=true;  
if(grid.map((r,i)=>r[4-i]).every(v=>v)) bingo=true;  
  
if(bingo){  
document.getElementById("bingoSound").play();  
db.ref("rooms/"+room+"/players/"+playerId+"/bingo").set(true);  
}  
}  
  
function updateRanking(players){  
let winners = Object.values(players)  
.filter(p=>p.bingo)  
.map(p=>p.name);  
  
document.getElementById("ranking").innerText =  
winners.join(" → ");  
}  
  
</script>  
</body>  
</html>  
