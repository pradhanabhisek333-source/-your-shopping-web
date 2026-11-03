
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Campus Marketplace</title>

<style>
*{box-sizing:border-box}
body{margin:0;font-family:Arial,sans-serif;background:#f3f5fa;color:#222}
header{background:#173b70;color:white;padding:20px;text-align:center}
.container{max-width:900px;margin:25px auto;padding:15px}
.card{background:white;padding:20px;margin:15px 0;border-radius:12px;box-shadow:0 2px 10px #ddd}
input,select,button,textarea{
width:100%;padding:12px;margin:8px 0;
border:1px solid #ccc;border-radius:7px;font-size:16px
}
button{background:#1769aa;color:white;border:0;cursor:pointer}
button:hover{background:#104d80}
.product{border:1px solid #ddd;padding:15px;margin:12px 0;border-radius:10px}
.product img{width:100%;max-height:220px;object-fit:contain}
.small{color:#666;font-size:13px}
.hidden{display:none}
a{color:#1769aa}
</style>
</head>

<body>

<header>
<h1>Campus Marketplace</h1>
<p>Buy • Sell • Rent • Connect</p>
</header>

<div class="container">

<section class="card" id="loginBox">
<h2>Student Login / Register</h2>
<p>Enter your details to access the marketplace.</p>

<input id="studentName" placeholder="Your full name">
<input id="phone" type="tel" placeholder="+91XXXXXXXXXX">

<button id="loginBtn">Enter Marketplace</button>
<p id="loginMessage" class="small"></p>
<p class="small">No OTP • No SMS • No reCAPTCHA</p>
</section>

<section class="card hidden" id="dashboard">
<h2>Welcome to Campus Marketplace!</h2>
<p id="userInfo"></p>
<p>You can both buy and sell products.</p>
<button id="logout">Logout</button>
</section>

<section class="card hidden" id="sellerBox">
<h2>Sell or Rent a Product</h2>

<input id="productName" placeholder="Product name">
<input id="productPrice" type="number" min="1" placeholder="Price in ₹">

<select id="productType">
<option value="Buy">For Sale</option>
<option value="Rent">For Rent</option>
</select>

<input id="productImage" placeholder="Image URL (optional)">

<textarea id="productDescription"
placeholder="Product description"
style="height:90px"></textarea>

<button id="addProduct">Publish Product</button>
<p id="sellerMessage" class="small"></p>
</section>

<section class="card hidden" id="marketBox">
<h2>Available Products</h2>
<div id="products">Loading products...</div>
</section>

</div>

<script type="module">

import {initializeApp} from
"https://www.gstatic.com/firebasejs/11.10.0/firebase-app.js";

import {
getAuth,
signInAnonymously,
onAuthStateChanged,
signOut
} from
"https://www.gstatic.com/firebasejs/11.10.0/firebase-auth.js";

import {
getFirestore,
doc,
setDoc,
getDoc,
addDoc,
collection,
onSnapshot,
serverTimestamp
} from
"https://www.gstatic.com/firebasejs/11.10.0/firebase-firestore.js";

// YOUR FIREBASE CONFIGURATION
const firebaseConfig = {
apiKey:"AIzaSyDQk5xe28AujBZMXnCu_KbY-TVHRktvsGI",
authDomain:"campus-marketplace-fd855.firebaseapp.com",
projectId:"campus-marketplace-fd855",
storageBucket:"campus-marketplace-fd855.firebasestorage.app",
messagingSenderId:"1012988673137",
appId:"1:1012988673137:web:7a3c4d2bc20eb898562fd0"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

let currentUser = null;
let studentProfile = null;
let unsubscribeProducts = null;

const $ = id => document.getElementById(id);

function message(id,text){
$(id).textContent=text;
}

// LOGIN WITHOUT OTP
$("loginBtn").addEventListener("click",async()=>{

const name=$("studentName").value.trim();
const phone=$("phone").value.trim();

if(name.length<2){
message("loginMessage","Please enter your name.");
return;
}

if(!/^\+[1-9]\d{7,14}$/.test(phone)){
message("loginMessage","Enter mobile number with country code, e.g. +91XXXXXXXXXX");
return;
}

$("loginBtn").disabled=true;
message("loginMessage","Logging in...");

try{

const result=await signInAnonymously(auth);
currentUser=result.user;

await setDoc(doc(db,"users",currentUser.uid),{
name:name,
phone:phone,
updatedAt:serverTimestamp()
},{merge:true});

studentProfile={name,phone};
showDashboard();

}catch(error){
message("loginMessage",error.message);
}else{
$("loginBtn").disabled=false;
}

});

// AUTH SESSION CHECK
onAuthStateChanged(auth,async(user)=>{

if(!user)return;

try{
currentUser=user;

const userDoc=await getDoc(doc(db,"users",user.uid));

if(userDoc.exists()){
studentProfile=userDoc.data();
showDashboard();
}else{
$("loginBox").classList.remove("hidden");
}

}catch(error){
message("loginMessage",error.message);
}

});

// SHOW DASHBOARD
function showDashboard(){

if(!currentUser || !studentProfile)return;

$("loginBox").classList.add("hidden");
$("dashboard").classList.remove("hidden");
$("sellerBox").classList.remove("hidden");
$("marketBox").classList.remove("hidden");

$("userInfo").textContent=
"Student: "+studentProfile.name+
" | Mobile: "+studentProfile.phone;

loadProducts();
}

// PUBLISH PRODUCT
$("addProduct").addEventListener("click",async()=>{

if(!currentUser || !studentProfile)return;

const name=$("productName").value.trim();
const price=Number($("productPrice").value);
const description=$("productDescription").value.trim();
const imageUrl=$("productImage").value.trim();
const type=$("productType").value;

if(!name || !Number.isFinite(price) || price<=0){
message("sellerMessage","Enter product name and valid price.");
return;
}

try{

await addDoc(collection(db,"products"),{
name:name,
price:price,
type:type,
imageUrl:imageUrl,
description:description,
sellerId:currentUser.uid,
sellerName:studentProfile.name,
sellerPhone:studentProfile.phone,
createdAt:serverTimestamp(),
isActive:true
});

message("sellerMessage","Product published successfully!");

$("productName").value="";
$("productPrice").value="";
$("productImage").value="";
$("productDescription").value="";

}catch(error){
message("sellerMessage",error.message);
}

});

// LOAD PRODUCTS
function loadProducts(){

if(unsubscribeProducts)unsubscribeProducts();

unsubscribeProducts=onSnapshot(
collection(db,"products"),
snapshot=>{

const area=$("products");
area.innerHTML="";

const available=snapshot.docs.filter(
d=>d.data().isActive!==false
);

if(available.length===0){
area.textContent="No products listed yet.";
return;
}

available.forEach(productDoc=>{

const p=productDoc.data();

const item=document.createElement("div");
item.className="product";

const title=document.createElement("h3");
title.textContent=p.name||"Product";

const price=document.createElement("p");
price.textContent="₹"+(p.price??"Ask seller")+
" • "+(p.type||"Sale");

const description=document.createElement("p");
description.textContent=p.description||"";

const seller=document.createElement("p");
seller.textContent="Seller: "+(p.sellerName||"Student");

item.append(title,price,description,seller);

if(p.imageUrl){

const img=document.createElement("img");
img.src=p.imageUrl;
img.alt=p.name||"Product image";
img.onerror=()=>img.remove();
item.prepend(img);

}

if(p.sellerPhone){

const contact=document.createElement("a");

contact.href="https://wa.me/"+
p.sellerPhone.replace(/\D/g,"");

contact.target="_blank";
contact.rel="noopener noreferrer";
contact.textContent="Contact seller on WhatsApp";

item.append(contact);

}

area.append(item);

});

},
error=>{
$("products").textContent=
"Could not load products: "+error.message;
}
);

}

// LOGOUT
$("logout").addEventListener("click",async()=>{

try{
if(unsubscribeProducts)unsubscribeProducts();
await signOut(auth);
location.reload();
}catch(error){
alert(error.message);
}

});

</script>
</body>
</html>
