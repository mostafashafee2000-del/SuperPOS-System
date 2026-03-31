# SuperPOS-System
Complete POS system
const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");

const app = express();
app.use(express.json());
app.use(cors());

mongoose.connect("mongodb://127.0.0.1/pos");

const Product = mongoose.model("Product", {
  name:String,
  price:Number,
  barcode:String,
  quantity:Number
});

const Sale = mongoose.model("Sale", {
  items:Array,
  total:Number,
  payment:String,
  customer:String,
  date:Date
});

const Customer = mongoose.model("Customer", {
  name:String,
  phone:String
});

let users = [
  {username:"admin", password:"1234", role:"admin"},
  {username:"cashier", password:"1111", role:"cashier"}
];

app.post("/login", (req,res)=>{
  let {username,password} = req.body;
  let user = users.find(u=>u.username===username && u.password===password);
  if(user) res.json(user);
  else res.status(401).json({message:"error"});
});

app.get("/products", async (req,res)=>{
  res.json(await Product.find());
});

app.post("/add-product", async (req,res)=>{
  await Product.create(req.body);
  res.json({message:"added"});
});

app.post("/sale", async (req,res)=>{
  let {items,total,payment,customer} = req.body;

  for(let item of items){
    let p = await Product.findOne({barcode:item.barcode});
    if(p){
      p.quantity -= item.qty;
      await p.save();
    }
  }

  await Sale.create({
    items,total,payment,customer,date:new Date()
  });

  res.json({message:"done"});
});

app.get("/reports", async (req,res)=>{
  res.json(await Sale.find());
});

app.get("/invoices", async (req,res)=>{
  res.json(await Sale.find());
});

app.post("/add-customer", async (req,res)=>{
  await Customer.create(req.body);
  res.json({message:"added"});
});

app.get("/customers", async (req,res)=>{
  res.json(await Customer.find());
});

app.listen(process.env.PORT || 3000, ()=> console.log("Server Running"));<h2>Login</h2>

<input id="u" placeholder="username">
<input id="p" type="password" placeholder="password">

<button onclick="login()">دخول</button>

<script>
async function login(){
  let res = await fetch("https://YOUR_RENDER_URL/sale/login",{
    method:"POST",
    headers:{"Content-Type":"application/json"},
    body: JSON.stringify({
      username:u.value,
      password:p.value
    })
  });

  if(res.ok){
    let user = await res.json();
    localStorage.setItem("user", JSON.stringify(user));
    if(user.role=="admin") location="admin.html";
    else location="index.html";
  }else alert("خطأ");
}
</script><body>

<input id="barcode" placeholder="Scan" onkeyup="scan(event)">
<input id="customer" placeholder="Customer">
<input id="discount" placeholder="Discount">

<select id="payment">
  <option value="cash">Cash</option>
  <option value="card">Card</option>
</select>

<div id="products"></div>

<h2>Invoice</h2>
<div id="cart"></div>
<h3 id="total"></h3>

<button onclick="checkout()">Pay</button>

<script>
let products=[], cart=[];

async function load(){
 let res = await fetch("https://YOUR_RENDER_URL/products");
 products = await res.json();

 let html="";
 products.forEach(p=>{
  html+=`<div onclick='add(${JSON.stringify(p)})'>
  ${p.name} - ${p.price}
  </div>`;
 });

 document.getElementById("products").innerHTML = html;
}

function add(p){
 let item = cart.find(x=>x.barcode===p.barcode);
 if(item) item.qty++;
 else cart.push({...p, qty:1});
 render();
}

function scan(e){
 if(e.key==="Enter"){
  let code = e.target.value;
  let p = products.find(x=>x.barcode==code);
  if(p) add(p);
  e.target.value="";
 }
}

function render(){
 let total=0, html="";
 cart.forEach(c=>{
  total+=c.price*c.qty;
  html+=`${c.name} x${c.qty}<br>`;
 });

 let discount = Number(document.getElementById("discount").value)||0;
 total-=discount;

 document.getElementById("cart").innerHTML=html;
 document.getElementById("total").innerText=total;
}

async function checkout(){
 await fetch("https://YOUR_RENDER_URL/sale",{
  method:"POST",
  headers:{"Content-Type":"application/json"},
  body: JSON.stringify({
    items:cart,
    total:document.getElementById("total").innerText,
    payment:payment.value,
    customer:customer.value
  })
 });

 window.print();
 cart=[];
 render();
}

load();
</script>

</body><h2>إضافة منتج</h2>

<input id="name" placeholder="Name">
<input id="price" placeholder="Price">
<input id="barcode" placeholder="Barcode">
<input id="qty" placeholder="Quantity">

<button onclick="addProduct()">Add</button>

<script>
async function addProduct(){
 await fetch("https://YOUR_RENDER_URL/add-product",{
  method:"POST",
  headers:{"Content-Type":"application/json"},
  body: JSON.stringify({
    name:name.value,
    price:Number(price.value),
    barcode:barcode.value,
    quantity:Number(qty.value)
  })
 });

 alert("Added");
}
</script><h2>Reports</h2>
<div id="data"></div>

<script>
async function load(){
 let res = await fetch("https://YOUR_RENDER_URL/reports");
 let sales = await res.json();

 let total=0;
 sales.forEach(s=> total+=Number(s.total));

 data.innerHTML = "Total Sales: " + total;
}
load();
</script><h2>Invoices</h2>
<div id="list"></div>

<script>
async function load(){
 let res = await fetch("https://YOUR_RENDER_URL/invoices");
 let data = await res.json();

 let html="";
 data.forEach(i=>{
  html+=`Invoice: ${i.total} - ${i.payment}<br>`;
 });

 list.innerHTML = html;
}
load();
</script>
