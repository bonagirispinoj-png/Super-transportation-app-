<!DOCTYPE html>
<html>
<head>
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Super Transport PRO++</title>
<style>
body{font-family:Arial;text-align:center;background:#eef2f5}
.box{background:white;padding:15px;margin:10px;border-radius:10px}
input,select,button{margin:5px;padding:8px;width:90%}
</style>
</head>
<body>

<h2>🚀 Super Transport PRO++</h2>

<div class="box" id="authBox">
  <input id="email" placeholder="Email">
  <input id="password" type="password" placeholder="Password">
  <select id="role">
    <option value="user">User</option>
    <option value="driver">Driver</option>
    <option value="admin">Admin</option>
  </select>
  <button onclick="signup()">Signup</button>
  <button onclick="login()">Login</button>
</div>

<div id="app"></div>

<script type="module">

import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
import {
  getFirestore, collection, addDoc, onSnapshot, updateDoc,
  doc, query, where, runTransaction, setDoc, getDoc
} from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
import {
  getAuth, createUserWithEmailAndPassword,
  signInWithEmailAndPassword, signOut
} from "https://www.gstatic.com/firebasejs/10.7.1/firebase-auth.js";

// 🔑 YOUR CONFIG — replace these with your actual Firebase project values
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  projectId: "YOUR_PROJECT"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);

let currentUser = {};
let unsubscribe = null;
let userLocation = { lat: null, lng: null }; // FIX 1: null so we know if GPS hasn't resolved yet

// 📍 GPS — FIX 1: added error handler
navigator.geolocation.watchPosition(
  pos => {
    userLocation = {
      lat: pos.coords.latitude,
      lng: pos.coords.longitude
    };
  },
  err => {
    console.warn("GPS unavailable:", err.message);
    // userLocation stays null — handled in book()
  }
);

// 🚪 LOGOUT
window.logout = async function () {
  if (unsubscribe) unsubscribe();
  await signOut(auth);
  location.reload();
};

// SIGNUP + ROLE SAVE
window.signup = async function () {
  try {
    const e = document.getElementById("email").value.trim();
    const p = document.getElementById("password").value;
    const r = document.getElementById("role").value;

    if (!e || !p) return alert("Fill all fields");

    let userCred = await createUserWithEmailAndPassword(auth, e, p);

    await setDoc(doc(db, "users", userCred.user.uid), {
      email: e,
      role: r,
      uid: userCred.user.uid  // FIX 3: store uid too
    });

    alert("Signup success! Please log in.");
  } catch (err) {
    alert(err.message);
  }
};

// LOGIN + FETCH ROLE
window.login = async function () {
  try {
    const e = document.getElementById("email").value.trim();
    const p = document.getElementById("password").value;

    let userCred = await signInWithEmailAndPassword(auth, e, p);
    let uid = userCred.user.uid;

    let snap = await getDoc(doc(db, "users", uid));
    if (!snap.exists()) return alert("User role not found. Please sign up first.");

    currentUser = { ...snap.data(), uid }; // FIX 3: always include uid

    document.getElementById("authBox").style.display = "none";

    if (unsubscribe) unsubscribe();

    if (currentUser.role === "user") userUI();
    else if (currentUser.role === "driver") driverUI();
    else if (currentUser.role === "admin") adminUI();
    else alert("Unknown role: " + currentUser.role);

  } catch (err) {
    alert(err.message);
  }
};

// 📏 DISTANCE CALCULATION (Haversine)
function getDistance(lat1, lng1, lat2, lng2) {
  const R = 6371;
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLng = (lng2 - lng1) * Math.PI / 180;
  const a =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
    Math.sin(dLng / 2) ** 2;
  return R * (2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a)));
}

// 💰 FARE — FIX 2: use actual GPS distance if available, fallback to default 3km
function calculateFare(vehicle, destLat, destLng) {
  const base = { Bike: 50, Auto: 80, Car: 120, Tractor: 300, JCB: 800, Lorry: 1500 };
  let distance = 3; // default fallback km
  if (userLocation.lat !== null && destLat && destLng) {
    distance = getDistance(userLocation.lat, userLocation.lng, destLat, destLng);
    if (distance < 0.5) distance = 0.5; // minimum 0.5km
  }
  return Math.floor((base[vehicle] || 100) + distance * 20);
}

// USER UI
function userUI() {
  document.getElementById("app").innerHTML = `
    <div class="box">
      <h3>👤 User Panel</h3>
      <button onclick="logout()">Logout</button>
      <select id="vehicle">
        <option>Bike</option><option>Auto</option><option>Car</option>
        <option>Tractor</option><option>JCB</option><option>Lorry</option>
      </select>
      <input id="pickup" placeholder="Pickup location">
      <input id="drop" placeholder="Drop location">
      <button onclick="book()">📦 Book Ride</button>
    </div>
    <div id="rides"></div>
  `;

  const q = query(collection(db, "rides"), where("uid", "==", currentUser.uid)); // FIX 3: query by uid not email

  unsubscribe = onSnapshot(q, snap => {
    let html = "";
    snap.forEach(d => {
      let data = d.data();
      html += `<div class="box"><p>🚗 ${data.vehicle} | ₹${data.fare} | 📍 ${data.pickup} → ${data.drop} | Status: <b>${data.status}</b></p></div>`;
    });
    document.getElementById("rides").innerHTML = html || "<p>No rides yet.</p>";
  });
}

// BOOK
window.book = async function () {
  try {
    let vehicle = document.getElementById("vehicle").value;
    let pickup = document.getElementById("pickup").value.trim();
    let drop = document.getElementById("drop").value.trim();

    if (!pickup || !drop) return alert("Enter both pickup and drop locations");

    // FIX 1: warn if GPS not available but still allow booking
    if (userLocation.lat === null) {
      alert("GPS location unavailable. Fare will use default distance.");
    }

    let fare = calculateFare(vehicle); // FIX 2: uses GPS coords if available

    await addDoc(collection(db, "rides"), {
      uid: currentUser.uid,       // FIX 3: store uid for secure querying
      user: currentUser.email,
      vehicle,
      pickup,
      drop,
      fare,
      status: "pending",
      userLat: userLocation.lat,
      userLng: userLocation.lng,
      createdAt: new Date().toISOString()
    });

    alert(`Ride booked! Estimated fare: ₹${fare}`);
  } catch (err) {
    alert(err.message);
  }
};

// DRIVER UI
function driverUI() {
  document.getElementById("app").innerHTML = `
    <div class="box">
      <h3>🚗 Driver Panel</h3>
      <button onclick="logout()">Logout</button>
    </div>
    <div id="rides"></div>
  `;

  const q = query(collection(db, "rides"), where("status", "in", ["pending", "accepted"]));

  unsubscribe = onSnapshot(q, snap => {
    let html = "";
    snap.forEach(d => {
      let data = d.data();
      let id = d.id;

      if (data.status === "accepted" && data.driver !== currentUser.email) return;

      if (data.status === "pending") {
        html += `<div class="box"><p>🚗 ${data.vehicle} | ₹${data.fare} | ${data.pickup} → ${data.drop}
          <button onclick="accept('${id}')">✅ Accept</button></p></div>`;
      }

      if (data.status === "accepted") {
        html += `<div class="box"><p>✅ Ongoing: ${data.vehicle} | ${data.pickup} → ${data.drop}
          <button onclick="complete('${id}')">🏁 Complete</button></p></div>`;
      }
    });
    document.getElementById("rides").innerHTML = html || "<p>No available rides.</p>";
  });
}

// ACCEPT — FIX 5: throw Error not plain string
window.accept = async function (id) {
  try {
    await runTransaction(db, async (t) => {
      let ref = doc(db, "rides", id);
      let ride = await t.get(ref);

      if (ride.data().status !== "pending") throw new Error("Ride already taken by another driver");

      t.update(ref, {
        status: "accepted",
        driver: currentUser.email
      });
    });
  } catch (err) {
    alert(err.message);
  }
};

// COMPLETE — FIX 6: only open UPI if update succeeds
window.complete = async function (id) {
  try {
    await updateDoc(doc(db, "rides", id), { status: "completed" });
    // Only open payment after successful update
    window.open("upi://pay?pa=bonagirispinoj-1@oksbi&pn=Transport");
  } catch (err) {
    alert("Failed to complete ride: " + err.message);
  }
};

// ADMIN UI — FIX 8: skip cancelled rides
function adminUI() {
  document.getElementById("app").innerHTML = `
    <div class="box">
      <h3>🛠 Admin Panel</h3>
      <button onclick="logout()">Logout</button>
    </div>
    <div id="rides"></div>
  `;

  unsubscribe = onSnapshot(collection(db, "rides"), snap => {
    let html = "";
    snap.forEach(d => {
      let data = d.data();
      let id = d.id;

      if (data.status === "cancelled") return; // FIX 8: don't show cancelled

      html += `<div class="box"><p>
        👤 ${data.user} | 🚗 ${data.vehicle} | ₹${data.fare} | <b>${data.status}</b>
        <button onclick="cancelRide('${id}')">❌ Cancel</button>
      </p></div>`;
    });
    document.getElementById("rides").innerHTML = html || "<p>No active rides.</p>";
  });
}

// CANCEL (renamed to avoid conflict with built-in cancel)
window.cancelRide = async function (id) {
  try {
    await updateDoc(doc(db, "rides", id), { status: "cancelled" });
  } catch (err) {
    alert(err.message);
  }
};

</script>
</body>
</html>
