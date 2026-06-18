<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue';

// --- FIREBASE CONFIGURATIE ---
import { initializeApp } from "firebase/app";
import { getFirestore, collection, getDocs, doc, setDoc, getDoc } from "firebase/firestore";

const firebaseConfig = {
  apiKey: "AIzaSyClVxQVctw6f-ba2GPPJ8rGO12M0V3zrgI",
  authDomain: "mallorca-62149.firebaseapp.com",
  projectId: "mallorca-62149",
  storageBucket: "mallorca-62149.firebasestorage.app",
  messagingSenderId: "345914520537",
  appId: "1:345914520537:web:b643569005fc8837f8dfc8"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);

const huidigScherm = ref('home'); 
const toonVerplaatsMenu = ref(false);
const toonNieuwMenu = ref(false);

// --- 1. AFTELKLOK LOGICA ---
const dagenTeGaan = ref(0);
const urenTeGaan = ref(0);
const minutenTeGaan = ref(0);
const secondenTeGaan = ref(0);
const aftelModus = ref('vertrek'); 
let aftelTimer: number;

const startCountdown = () => {
  const vertrekDatum = new Date('2026-07-08T08:30:00').getTime();
  const eindDatum = new Date('2026-07-17T18:45:00').getTime();

  const berekenTijd = (verschil: number) => {
    dagenTeGaan.value = Math.floor(verschil / (1000 * 60 * 60 * 24));
    urenTeGaan.value = Math.floor((verschil % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
    minutenTeGaan.value = Math.floor((verschil % (1000 * 60 * 60)) / (1000 * 60));
    secondenTeGaan.value = Math.floor((verschil % (1000 * 60)) / 1000);
  };

  const updateTimer = () => {
    const nu = new Date().getTime();
    const verschilVertrek = vertrekDatum - nu;
    const verschilEinde = eindDatum - nu;
    
    if (verschilVertrek > 0) {
      aftelModus.value = 'vertrek';
      berekenTijd(verschilVertrek);
    } else if (verschilEinde > 0) {
      aftelModus.value = 'terugreis';
      berekenTijd(verschilEinde);
    } else {
      aftelModus.value = 'voorbij';
      clearInterval(aftelTimer);
    }
  };
  
  updateTimer(); 
  aftelTimer = setInterval(updateTimer, 1000);
};

// --- 2. LIVE WEER LOGICA ---
const liveWeer = ref({
  calaRatjada: { temp: '--', conditie: '', icoon: '⏳' },
  haren: { temp: '--', conditie: '', icoon: '⏳' }
});

const voorspelling = ref<any[]>([]);

const haalWeerOp = async () => {
  const apiKey = '26a57f31f3ae779576631ba7e3ac4894';
  const getIcoon = (iconCode: string) => {
    const map: Record<string, string> = {
      '01d': '☀️', '01n': '🌙', '02d': '⛅', '02n': '☁️',
      '03d': '☁️', '03n': '☁️', '04d': '☁️', '04n': '☁️',
      '09d': '🌧️', '09n': '🌧️', '10d': '🌦️', '10n': '🌧️',
      '11d': '⛈️', '11n': '⛈️', '13d': '❄️', '13n': '❄️',
      '50d': '🌫️', '50n': '🌫️'
    };
    return map[iconCode] || '🌤️';
  };

  try {
    const resHaren = await fetch(`https://api.openweathermap.org/data/2.5/weather?q=Haren,nl&units=metric&lang=nl&appid=${apiKey}`);
    const dataHaren = await resHaren.json();
    if(dataHaren.main) {
      liveWeer.value.haren.temp = Math.round(dataHaren.main.temp).toString();
      liveWeer.value.haren.conditie = dataHaren.weather[0].description;
      liveWeer.value.haren.icoon = getIcoon(dataHaren.weather[0].icon);
    }

    const resMallorca = await fetch(`https://api.openweathermap.org/data/2.5/weather?q=Capdepera,es&units=metric&lang=nl&appid=${apiKey}`);
    const dataMallorca = await resMallorca.json();
    if(dataMallorca.main) {
      liveWeer.value.calaRatjada.temp = Math.round(dataMallorca.main.temp).toString();
      liveWeer.value.calaRatjada.conditie = dataMallorca.weather[0].description;
      liveWeer.value.calaRatjada.icoon = getIcoon(dataMallorca.weather[0].icon);
    }

    const resForecast = await fetch(`https://api.openweathermap.org/data/2.5/forecast?q=Capdepera,es&units=metric&lang=nl&appid=${apiKey}`);
    const dataForecast = await resForecast.json();
    if (dataForecast.list) {
      const middagVoorspellingen = dataForecast.list.filter((item: any) => item.dt_txt.includes('12:00:00'));
      voorspelling.value = middagVoorspellingen.slice(0, 5).map((item: any) => {
        const datum = new Date(item.dt * 1000);
        const dagNaam = datum.toLocaleDateString('nl-NL', { weekday: 'long' });
        return {
          dag: dagNaam.charAt(0).toUpperCase() + dagNaam.slice(1), 
          temp: Math.round(item.main.temp),
          icoon: getIcoon(item.weather[0].icon),
          omschrijving: item.weather[0].description
        };
      });
    }
  } catch (error) { console.error("Weer fout:", error); }
};

const heeftSlechtWeerWaarschuwing = (omschrijving: string) => {
  const huidigWeer = liveWeer.value.calaRatjada.conditie.toLowerCase();
  const slechteCondities = ['regen', 'rain', 'onweer', 'storm', 'thunder', 'bui'];
  const isSlechtWeer = slechteCondities.some(w => huidigWeer.includes(w));
  const buitenActiviteiten = ['strand', 'zwembad', 'zee', 'snorkelen', 'fietsen', 'duik', 'kustpad'];
  const isBuiten = buitenActiviteiten.some(b => omschrijving.toLowerCase().includes(b));

  return isSlechtWeer && isBuiten;
};

// --- 3 & 4. FIREBASE SYNCHRONISATIE (GIDS & DAGSCHEMA) ---
const gidsItems = ref<any[]>([]);
const laadtGids = ref(true);
const dagschema = ref<any[]>([]);
const laadtSchema = ref(true);

const standaardGids = [
  { id: 1, categorie: 'Strand & snorkelen', naam: 'Cala Gat', reistijd: '5 min lopen', eigenschap: 'Klein, rustig, geweldig om te snorkelen', icoon: '🤿' },
  { id: 2, categorie: 'Strand & snorkelen', naam: 'Cala Lliteras', reistijd: '10 min lopen', eigenschap: 'Kleine rotsbaai, prachtig water', icoon: '🐟' },
  { id: 3, categorie: 'Strand & snorkelen', naam: 'Cala Mesquida', reistijd: '10 min auto', eigenschap: 'Groot zandstrand, mooie golven', icoon: '🌊' },
  { id: 4, categorie: 'Eten & drinken', naam: 'Voorbeeld Restaurant', reistijd: '15 min', eigenschap: 'Lekkere tapas aan zee', icoon: '🥘' }
];

const standaardDagschema = [
  { id: 1, datum: 'Woensdag 8 juli 2026', titel: 'Woensdag 8 juli', activiteiten: [ { id: 101, tijd: '08:30', omschrijving: 'Vertrek auto Haren', icoon: '🚗', geselecteerd: false, voltooid: false }, { id: 102, tijd: '10:30', omschrijving: 'Aankomst FMO', icoon: '🅿️', geselecteerd: false, voltooid: false }, { id: 103, tijd: '12:50', omschrijving: 'Vlucht FR7386', icoon: '✈️', geselecteerd: false, voltooid: false }, { id: 104, tijd: '15:10', omschrijving: 'Aankomst Palma', icoon: '🛬', geselecteerd: false, voltooid: false }, { id: 105, tijd: '16:00', omschrijving: 'Huurauto ophalen', icoon: '🚙', geselecteerd: false, voltooid: false }, { id: 106, tijd: '17:15', omschrijving: 'Inchecken hotel', icoon: '🏨', geselecteerd: false, voltooid: false }, { id: 107, tijd: '19:30', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false, voltooid: false } ] },
  { id: 2, datum: 'Donderdag 9 juli 2026', titel: 'Donderdag 9 juli', activiteiten: [ { id: 201, tijd: '09:30', omschrijving: 'Rustig ontbijten', icoon: '☕', geselecteerd: false, voltooid: false }, { id: 202, tijd: '11:00', omschrijving: 'Chillen zwembad', icoon: '🌊', geselecteerd: false, voltooid: false }, { id: 203, tijd: '13:30', omschrijving: 'Snelle lunch', icoon: '🍴', geselecteerd: false, voltooid: false }, { id: 204, tijd: '14:30', omschrijving: 'Duik Cala Agulla', icoon: '🏖️', geselecteerd: false, voltooid: false }, { id: 205, tijd: '19:00', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false, voltooid: false } ] },
  { id: 3, datum: 'Vrijdag 10 juli 2026', titel: 'Vrijdag 10 juli', activiteiten: [ { id: 301, tijd: '09:00', omschrijving: 'Gezamenlijk ontbijt', icoon: '☕', geselecteerd: false, voltooid: false }, { id: 302, tijd: '10:30', omschrijving: 'Rit Cala Mesquida', icoon: '🚙', geselecteerd: false, voltooid: false }, { id: 303, tijd: '11:00', omschrijving: 'Wandeling kustpad', icoon: '🚶', geselecteerd: false, voltooid: false }, { id: 304, tijd: '12:00', omschrijving: 'Snorkelen baai', icoon: '🐟', geselecteerd: false, voltooid: false }, { id: 305, tijd: '14:00', omschrijving: 'Lichte lunch', icoon: '🥗', geselecteerd: false, voltooid: false }, { id: 306, tijd: '15:30', omschrijving: 'Zwembad', icoon: '🌊', geselecteerd: false, voltooid: false }, { id: 307, tijd: '19:30', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false, voltooid: false } ] },
  { id: 4, datum: 'Zaterdag 11 juli 2026', titel: 'Zaterdag 11 juli', activiteiten: [ { id: 401, tijd: '08:00', omschrijving: 'Racefiets ophalen (Ewout)', icoon: '🚲', geselecteerd: false, voltooid: false }, { id: 402, tijd: '08:30', omschrijving: 'Bergrit Artà (Ewout)', icoon: '⛰️', geselecteerd: false, voltooid: false }, { id: 403, tijd: '08:30', omschrijving: 'Uitslapen (Vera & Tris)', icoon: '😴', geselecteerd: false, voltooid: false }, { id: 404, tijd: '11:30', omschrijving: 'Fiets inleveren (Ewout)', icoon: '🚲', geselecteerd: false, voltooid: false }, { id: 405, tijd: '13:30', omschrijving: 'Lunch', icoon: '🍴', geselecteerd: false, voltooid: false }, { id: 406, tijd: '15:00', omschrijving: 'Cala Gat', icoon: '🏖️', geselecteerd: false, voltooid: false }, { id: 407, tijd: '19:00', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false, voltooid: false } ] },
  { id: 5, datum: 'Zondag 12 juli 2026', titel: 'Zondag 12 juli', activiteiten: [ { id: 501, tijd: '09:30', omschrijving: 'Ontbijt', icoon: '☕', geselecteerd: false, voltooid: false }, { id: 502, tijd: '11:00', omschrijving: 'Rit strandclub', icoon: '🍹', geselecteerd: false, voltooid: false }, { id: 503, tijd: '13:00', omschrijving: 'Tapas aan zee', icoon: '🥘', geselecteerd: false, voltooid: false }, { id: 504, tijd: '15:30', omschrijving: 'Ontspanning hotel', icoon: '😎', geselecteerd: false, voltooid: false }, { id: 505, tijd: '19:30', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false, voltooid: false } ] },
  { id: 6, datum: 'Maandag 13 juli 2026', titel: 'Maandag 13 juli', activiteiten: [ { id: 601, tijd: '09:00', omschrijving: 'Ontbijten', icoon: '☕', geselecteerd: false, voltooid: false }, { id: 602, tijd: '10:00', omschrijving: 'Winkelen stad', icoon: '🛍️', geselecteerd: false, voltooid: false }, { id: 603, tijd: '14:00', omschrijving: 'Late lunch', icoon: '🌮', geselecteerd: false, voltooid: false }, { id: 604, tijd: '15:30', omschrijving: 'Terugrit', icoon: '🚙', geselecteerd: false, voltooid: false }, { id: 605, tijd: '17:00', omschrijving: 'Zwembad', icoon: '🌊', geselecteerd: false, voltooid: false }, { id: 606, tijd: '19:30', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false, voltooid: false } ] },
  { id: 7, datum: 'Dinsdag 14 juli 2026', titel: 'Dinsdag 14 juli', activiteiten: [ { id: 701, tijd: '09:30', omschrijving: 'Ontbijt', icoon: '☕', geselecteerd: false, voltooid: false }, { id: 702, tijd: '11:00', omschrijving: 'Snorkelen Lliteras', icoon: '🐟', geselecteerd: false, voltooid: false }, { id: 703, tijd: '13:30', omschrijving: 'Lichte lunch', icoon: '🍴', geselecteerd: false, voltooid: false }, { id: 704, tijd: '14:30', omschrijving: 'Lezen bij hotel', icoon: '📖', geselecteerd: false, voltooid: false }, { id: 705, tijd: '19:00', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false, voltooid: false } ] },
  { id: 8, datum: 'Woensdag 15 juli 2026', titel: 'Woensdag 15 juli', activiteiten: [ { id: 801, tijd: '09:30', omschrijving: 'Ontbijt', icoon: '☕', geselecteerd: false, voltooid: false }, { id: 802, tijd: '11:00', omschrijving: 'Vuurtoren wandeling', icoon: '🗼', geselecteerd: false, voltooid: false }, { id: 803, tijd: '13:00', omschrijving: 'Lunch', icoon: '🥗', geselecteerd: false, voltooid: false }, { id: 804, tijd: '14:00', omschrijving: 'Zwemmen', icoon: '🌊', geselecteerd: false, voltooid: false }, { id: 805, tijd: '19:30', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false, voltooid: false } ] },
  { id: 9, datum: 'Donderdag 16 juli 2026', titel: 'Donderdag 16 juli', activiteiten: [ { id: 901, tijd: '09:30', omschrijving: 'Ontbijt', icoon: '☕', geselecteerd: false, voltooid: false }, { id: 902, tijd: '11:00', omschrijving: 'Ontspanning zwembad', icoon: '😎', geselecteerd: false, voltooid: false }, { id: 903, tijd: '13:30', omschrijving: 'Lunch', icoon: '🍴', geselecteerd: false, voltooid: false }, { id: 904, tijd: '14:30', omschrijving: 'Souvenirs scoren', icoon: '🛍️', geselecteerd: false, voltooid: false }, { id: 905, tijd: '19:00', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false, voltooid: false } ] },
  { id: 10, datum: 'Vrijdag 17 juli 2026', titel: 'Vrijdag 17 juli', activiteiten: [ { id: 1001, tijd: '09:00', omschrijving: 'Koffers sluiten', icoon: '🎒', geselecteerd: false, voltooid: false }, { id: 1002, tijd: '10:00', omschrijving: 'Uitchecken', icoon: '🏨', geselecteerd: false, voltooid: false }, { id: 1003, tijd: '10:30', omschrijving: 'Rit vliegveld', icoon: '🚙', geselecteerd: false, voltooid: false }, { id: 1004, tijd: '11:45', omschrijving: 'Auto inleveren', icoon: '🔑', geselecteerd: false, voltooid: false }, { id: 1005, tijd: '13:45', omschrijving: 'Vlucht FR7282', icoon: '✈️', geselecteerd: false, voltooid: false }, { id: 1006, tijd: '16:25', omschrijving: 'Aankomst FMO', icoon: '🛬', geselecteerd: false, voltooid: false }, { id: 1007, tijd: '17:00', omschrijving: 'Rit naar huis', icoon: '🚗', geselecteerd: false, voltooid: false }, { id: 1008, tijd: '18:45', omschrijving: 'Aankomst Haren', icoon: '🏠', geselecteerd: false, voltooid: false } ] }
];

const gegroepeerdeGids = computed(() => {
  const groepen: Record<string, any[]> = {};
  gidsItems.value.forEach(item => {
    if (!groepen[item.categorie]) {
      groepen[item.categorie] = [];
    }
    groepen[item.categorie].push(item);
  });
  return groepen;
});

const slaSchemaLiveOp = async () => {
  try {
    await setDoc(doc(db, "appData", "dagschema"), { dagen: dagschema.value });
  } catch (error) { console.error("Fout bij opslaan schema:", error); }
};

const haalDataOp = async () => {
  try {
    const gidsDoc = await getDoc(doc(db, "appData", "gidsData"));
    if (gidsDoc.exists()) {
      gidsItems.value = gidsDoc.data().lijst;
    } else {
      await setDoc(doc(db, "appData", "gidsData"), { lijst: standaardGids });
      gidsItems.value = standaardGids;
    }
    
    const schemaDoc = await getDoc(doc(db, "appData", "dagschema"));
    if (schemaDoc.exists()) {
      dagschema.value = schemaDoc.data().dagen;
    } else {
      await setDoc(doc(db, "appData", "dagschema"), { dagen: standaardDagschema });
      dagschema.value = standaardDagschema;
    }
  } catch (error) {
    console.error("Firebase ophaalfout:", error);
    gidsItems.value = standaardGids;
    dagschema.value = standaardDagschema;
  } finally {
    laadtGids.value = false;
    laadtSchema.value = false;
  }
};

// --- START FUNCTIES ---
onMounted(() => {
  startCountdown();
  haalWeerOp();
  haalDataOp(); 
});

onUnmounted(() => { clearInterval(aftelTimer); });

// --- 5. INTERACTIES & FORMULIER LOGICA ---
const formTijd = ref('12:00');
const formOmschrijving = ref('');
const formIcoon = ref('📌');
const formDagId = ref(1);

const voegActiviteitToe = () => {
  if(!formOmschrijving.value) return;
  const doelDag = dagschema.value.find(dag => dag.id === formDagId.value);
  if (doelDag) {
    doelDag.activiteiten.push({
      id: Date.now(), 
      tijd: formTijd.value,
      omschrijving: formOmschrijving.value,
      icoon: formIcoon.value,
      geselecteerd: false,
      voltooid: false
    });
    doelDag.activiteiten.sort((a: any, b: any) => a.tijd.localeCompare(b.tijd));
    slaSchemaLiveOp(); 
    toonNieuwMenu.value = false;
    formOmschrijving.value = ''; 
  }
};

const toggleSelectie = (activiteit: any) => {
  activiteit.geselecteerd = !activiteit.geselecteerd;
};

const heeftSelectie = computed(() => {
  return dagschema.value.some(dag => dag.activiteiten.some((act: any) => act.geselecteerd));
});

const verwijderGeselecteerde = () => {
  dagschema.value.forEach(dag => {
    dag.activiteiten = dag.activiteiten.filter((act: any) => !act.geselecteerd);
  });
  slaSchemaLiveOp(); 
  toonVerplaatsMenu.value = false; 
};

const verplaatsGeselecteerdeNaar = (doelDagId: number) => {
  let teVerplaatsen: any[] = [];
  dagschema.value.forEach(dag => {
    const geselecteerd = dag.activiteiten.filter((act: any) => act.geselecteerd);
    teVerplaatsen.push(...geselecteerd);
    dag.activiteiten = dag.activiteiten.filter((act: any) => !act.geselecteerd);
  });

  const doelDag = dagschema.value.find(dag => dag.id === doelDagId);
  if (doelDag) {
    teVerplaatsen.forEach(act => {
      act.geselecteerd = false; 
      doelDag.activiteiten.push(act);
    });
    doelDag.activiteiten.sort((a: any, b: any) => a.tijd.localeCompare(b.tijd));
    slaSchemaLiveOp();
  }
  toonVerplaatsMenu.value = false;
};

const rondGeselecteerdeAf = () => {
  dagschema.value.forEach(dag => {
    dag.activiteiten.forEach((act: any) => {
      if(act.geselecteerd) {
        act.voltooid = !act.voltooid; 
        act.geselecteerd = false;     
      }
    });
  });
  slaSchemaLiveOp();
};
</script>

<template>
  <main class="app-container">
    <header class="app-header">
      <h1>Mallorca 2026</h1>
      <span class="header-icon">😎</span>
    </header>

    <nav class="hoofd-menu">
      <button :class="{ 'menu-actief': huidigScherm === 'home' }" @click="huidigScherm = 'home'">🏠 Home</button>
      <button :class="{ 'menu-actief': huidigScherm === 'planner' }" @click="huidigScherm = 'planner'">📅 Planner</button>
      <button :class="{ 'menu-actief': huidigScherm === 'gids' }" @click="huidigScherm = 'gids'">🏖️ Gids</button>
    </nav>

    <!-- Dynamische CSS class op basis van het huidige scherm -->
    <div class="content-gebied" :class="huidigScherm === 'home' ? 'home-layout' : 'scroll-layout'">
      <transition name="fade" mode="out-in">
        
        <!-- HOME SCHERM (Volledig beeldvullend zonder scroll) -->
        <div v-if="huidigScherm === 'home'" class="home-scherm">
          <h2 class="welkom-titel">Welkom Weiner dogs!</h2>
          
          <div class="compact-grid">
            <div class="compact-kaart aftel-kaart">
              <template v-if="aftelModus === 'vertrek'">
                <h4>⏳ Vertrek over:</h4>
                <div class="compact-groot">{{ dagenTeGaan }}d {{ urenTeGaan }}u</div>
                <div class="compact-klein">{{ minutenTeGaan }}m <span class="sec">{{ secondenTeGaan }}s</span></div>
              </template>
              <template v-else-if="aftelModus === 'terugreis'">
                <h4>🌴 Vakantie voorbij over:</h4>
                <div class="compact-groot">{{ dagenTeGaan }}d {{ urenTeGaan }}u</div>
                <div class="compact-klein">{{ minutenTeGaan }}m <span class="sec">{{ secondenTeGaan }}s</span></div>
              </template>
              <template v-else>
                <h4>🏠 Weer thuis!</h4>
                <div class="compact-klein">De vakantie is afgelopen.</div>
              </template>
            </div>

            <div class="compact-kaart actueel-weer-kaart">
              <h4>Actueel Weer</h4>
              <div class="weer-regel">
                <span class="weer-locatie">Cala Ratjada</span>
                <span class="weer-temp">{{ liveWeer.calaRatjada.icoon }} {{ liveWeer.calaRatjada.temp }}°C</span>
              </div>
              <div class="weer-regel">
                <span class="weer-locatie">Haren</span>
                <span class="weer-temp">{{ liveWeer.haren.icoon }} {{ liveWeer.haren.temp }}°C</span>
              </div>
            </div>
          </div>

          <div class="voorspelling-kaart">
            <h3>Weersverwachting Cala Ratjada</h3>
            <p class="voorspelling-sub">Komende 5 dagen</p>
            <div class="voorspelling-lijst">
              <div v-for="(dag, index) in voorspelling" :key="index" class="voorspelling-rij">
                <span class="v-dag">{{ dag.dag }}</span>
                <span class="v-icoon">{{ dag.icoon }}</span>
                <span class="v-temp">{{ dag.temp }}°C</span>
                <span class="v-desc">{{ dag.omschrijving }}</span>
              </div>
              <div v-if="voorspelling.length === 0" class="voorspelling-laden">Voorspelling laden...</div>
            </div>
          </div>

          <!-- NIEUW: Praktische Info Blokje -->
          <div class="praktische-info-kaart">
            <h3>ℹ️ Praktische Info</h3>
            <div class="info-rij">
              <span class="info-label">Vlucht Heen:</span>
              <span class="info-waarde">FR7386 (12:50)</span>
            </div>
            <div class="info-rij">
              <span class="info-label">Vlucht Terug:</span>
              <span class="info-waarde">FR7282 (13:45)</span>
            </div>
            <div class="info-rij">
              <span class="info-label">Auto inleveren:</span>
              <span class="info-waarde">17 juli (11:45)</span>
            </div>
          </div>

        </div>

        <!-- PLANNER SCHERM (Scrollen mogelijk) -->
        <div v-else-if="huidigScherm === 'planner'">
          <div v-if="laadtSchema" class="laad-scherm"><div class="spinner"></div><p>Schema laden...</p></div>
          <div v-else>
            <div v-for="dag in dagschema" :key="dag.id" class="dag-sectie">
              <h2 class="dag-titel">{{ dag.titel }}</h2>
              
              <transition-group name="lijst" tag="div" class="activiteiten-grid">
                <div 
                  v-for="activiteit in dag.activiteiten" 
                  :key="activiteit.id" 
                  class="tegel"
                  :class="{ 'tegel-actief': activiteit.geselecteerd, 'tegel-voltooid': activiteit.voltooid }"
                  @click="toggleSelectie(activiteit)"
                >
                  <div v-if="activiteit.geselecteerd" class="vinkje">✓</div>
                  <div v-if="heeftSlechtWeerWaarschuwing(activiteit.omschrijving)" class="weer-waarschuwing" title="Slecht weer voorspeld!">⚠️</div>
                  
                  <div class="icoon">{{ activiteit.icoon }}</div>
                  <div class="titel">{{ activiteit.omschrijving }}</div>
                  <div class="tijd">{{ activiteit.tijd }}</div>
                </div>
              </transition-group>
            </div>
          </div>
        </div>

        <!-- GIDS SCHERM (Scrollen mogelijk) -->
        <div v-else-if="huidigScherm === 'gids'">
          <h2 class="hoofd-titel-gids">Lokale gids</h2>
          <div v-if="laadtGids" class="laad-scherm"><div class="spinner"></div><p>Gids ophalen...</p></div>
          
          <div v-else>
            <div v-for="(items, categorie) in gegroepeerdeGids" :key="categorie" class="categorie-sectie">
              <h3 class="gids-categorie-titel">{{ categorie }}</h3>
              <transition-group name="lijst" tag="div" class="gids-lijst">
                <div v-for="item in items" :key="item.id" class="gids-kaart">
                  <div class="gids-icoon">{{ item.icoon }}</div>
                  <div class="gids-info">
                    <h3>{{ item.naam }}</h3>
                    <p class="gids-detail">⏱️ {{ item.reistijd }}</p>
                    <p class="gids-detail">✨ {{ item.eigenschap }}</p>
                  </div>
                </div>
              </transition-group>
            </div>
          </div>
        </div>

      </transition>
    </div>
    
    <!-- BOTTOM PANEL VOOR PLANNER -->
    <div class="bottom-panel" v-if="huidigScherm === 'planner' && !laadtSchema">
      <transition name="fade" mode="out-in">
        
        <div v-if="heeftSelectie" key="acties">
          <div v-if="!toonVerplaatsMenu">
            <h3>Geselecteerde acties</h3>
            <div class="actie-knoppen-rij">
              <button class="actie-knop afrond-knop" @click="rondGeselecteerdeAf">✅ Afronden</button>
              <button class="actie-knop verplaats-knop" @click="toonVerplaatsMenu = true">➡️ Verplaats</button>
              <button class="actie-knop verwijder-knop" @click="verwijderGeselecteerde">🗑️ Wis</button>
            </div>
          </div>
          <div v-else>
            <h3>Verplaatsen naar:</h3>
            <div class="kies-dag-lijst">
              <button v-for="dag in dagschema" :key="dag.id" class="kies-dag-knop" @click="verplaatsGeselecteerdeNaar(dag.id)">
                {{ dag.titel }}
              </button>
            </div>
            <button class="annuleer-knop" @click="toonVerplaatsMenu = false">✖ Annuleren</button>
          </div>
        </div>

        <div v-else-if="toonNieuwMenu" key="nieuw">
          <h3>Nieuwe activiteit toevoegen</h3>
          <div class="formulier">
            <select v-model="formDagId" class="form-input">
              <option v-for="dag in dagschema" :key="dag.id" :value="dag.id">{{ dag.titel }}</option>
            </select>
            <div class="form-rij">
              <input type="time" v-model="formTijd" class="form-input form-tijd" />
              <input type="text" v-model="formOmschrijving" placeholder="Wat ga je doen?" class="form-input form-tekst" />
            </div>
            <div class="actie-knoppen-rij">
              <button class="actie-knop afrond-knop" @click="voegActiviteitToe">Toevoegen</button>
              <button class="actie-knop verwijder-knop" @click="toonNieuwMenu = false">Annuleren</button>
            </div>
          </div>
        </div>

        <div v-else key="leeg">
          <h3>Wat ga je er doen?</h3>
          <p style="margin-bottom:10px;">Selecteer acties om ze te bewerken.</p>
          <button class="nieuwe-actie-knop" @click="toonNieuwMenu = true">➕ Nieuwe activiteit</button>
        </div>

      </transition>
    </div>
  </main>
</template>

<style>
:root {
  --teal-licht: #a0ceb9;
  --teal-donker: #7bb29e;
  --oranje-vinkje: #f39c12;
  --blauw-knop: #3498db;
  --rood-knop: #e74c3c;
  --groen-knop: #2ecc71;
  --achtergrond: #fdfdfd;
  --tekst-grijs: #7f8c8d;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background-color: #e8f0eb;
  margin: 0;
  padding: 0;
  display: flex;
  justify-content: center;
  overscroll-behavior-y: none; 
}

.app-container {
  width: 100%;
  max-width: 400px;
  background-color: var(--achtergrond);
  height: 100dvh; 
  display: flex;
  flex-direction: column;
  position: relative;
  box-shadow: 0 0 20px rgba(0,0,0,0.1);
  overflow-x: hidden;
}

.app-header, .hoofd-menu {
  flex-shrink: 0; 
}

.app-header {
  background-color: var(--teal-licht);
  color: white;
  padding: 15px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.app-header h1 { margin: 0; font-size: 1.4rem; font-weight: 600; text-align: left; }
.header-icon { font-size: 1.5rem; }

.hoofd-menu { display: flex; background-color: white; border-bottom: 2px solid #eee; }
.hoofd-menu button {
  flex: 1; padding: 12px 0; border: none; background: none; font-size: 0.9rem; color: var(--tekst-grijs); cursor: pointer; font-weight: bold; transition: background-color 0.2s;
}
.hoofd-menu button.menu-actief { color: var(--teal-donker); border-bottom: 3px solid var(--teal-donker); }

.content-gebied {
  flex: 1; 
  display: flex;
  flex-direction: column;
}

.home-layout {
  padding: 15px;
  overflow: hidden; 
}
.scroll-layout {
  padding: 20px 15px 180px 15px;
  overflow-y: auto; 
}

/* COMPACTERE HOME SCHERM LAYOUT */
.home-scherm {
  display: flex;
  flex-direction: column;
  height: 100%;
  justify-content: space-evenly; 
  gap: 6px; /* Aangepast van 15px naar 6px voor perfecte pasvorm */
}
.welkom-titel { margin-top: 0; margin-bottom: 0; color: var(--teal-donker); font-size: 1.15rem; text-align: center; } /* Iets kleiner */

.compact-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin-bottom: 0; } /* Iets compactere grid */
.compact-kaart { border-radius: 8px; padding: 8px; text-align: center; box-shadow: 0 2px 5px rgba(0,0,0,0.05); display: flex; flex-direction: column; justify-content: center; }
.compact-kaart h4 { margin: 0 0 6px 0; font-size: 0.85rem; font-weight: 600; }
.aftel-kaart { background-color: var(--teal-donker); color: white; }
.compact-groot { font-size: 1.1rem; font-weight: bold; }
.compact-klein { font-size: 0.85rem; margin-top: 2px; opacity: 0.9; }
.sec { font-size: 0.75rem; opacity: 0.8; }

.actueel-weer-kaart { background-color: white; border: 1px solid var(--teal-licht); color: var(--teal-donker); }
.weer-regel { display: flex; justify-content: space-between; align-items: center; margin: 2px 0; font-size: 0.85rem; }
.weer-locatie { color: var(--tekst-grijs); font-weight: 500; }
.weer-temp { font-weight: bold; }

/* COMPACTERE VOORSPELLING */
.voorspelling-kaart { background-color: white; border: 2px solid var(--teal-licht); border-radius: 8px; padding: 10px 15px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); margin-bottom: 0; }
.voorspelling-kaart h3 { margin: 0; color: var(--teal-donker); font-size: 1rem; }
.voorspelling-sub { color: var(--tekst-grijs); font-size: 0.75rem; margin: 2px 0 6px 0; border-bottom: 1px solid #eee; padding-bottom: 5px; }
.voorspelling-lijst { display: flex; flex-direction: column; gap: 6px; }
.voorspelling-rij { display: flex; align-items: center; justify-content: space-between; padding: 1px 0; border-bottom: 1px dashed #eee; }
.voorspelling-rij:last-child { border-bottom: none; }
.v-dag { font-weight: bold; color: #333; width: 80px; font-size: 0.85rem; }
.v-icoon { font-size: 1.2rem; }
.v-temp { font-weight: bold; color: var(--teal-donker); width: 40px; text-align: right; font-size: 0.85rem; }
.v-desc { font-size: 0.7rem; color: var(--tekst-grijs); text-transform: capitalize; text-align: right; flex: 1; }

/* COMPACTERE PRAKTISCHE INFO */
.praktische-info-kaart {
  background-color: white;
  border: 2px solid var(--teal-licht);
  border-radius: 8px;
  padding: 10px 15px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}
.praktische-info-kaart h3 {
  margin: 0 0 8px 0;
  color: var(--teal-donker);
  font-size: 1.05rem;
}
.info-rij {
  display: flex;
  justify-content: space-between;
  padding: 4px 0;
  border-bottom: 1px solid #f5f5f5;
  font-size: 0.85rem;
}
.info-rij:last-child { border-bottom: none; }
.info-label { color: var(--tekst-grijs); }
.info-waarde { font-weight: bold; color: #333; }

/* Planner */
.dag-titel { font-size: 1.1rem; color: var(--teal-donker); margin-bottom: 15px; border-bottom: 1px solid #eee; padding-bottom: 5px; }
.activiteiten-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; margin-bottom: 30px; position: relative; }
.tegel { background-color: white; border: 2px solid var(--teal-licht); border-radius: 4px; padding: 15px 5px; text-align: center; cursor: pointer; position: relative; display: flex; flex-direction: column; align-items: center; justify-content: center; min-height: 90px; transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1); }
.tegel:active { transform: scale(0.92); }
.icoon { font-size: 2rem; margin-bottom: 8px; }
.titel { color: var(--teal-donker); font-size: 0.8rem; font-weight: 600; line-height: 1.2; }
.tijd { color: var(--tekst-grijs); font-size: 0.7rem; margin-top: 4px; }

.tegel-actief { background-color: var(--teal-donker); border-color: var(--teal-donker); }
.tegel-actief .titel, .tegel-actief .tijd { color: white; }

.tegel-voltooid { opacity: 0.5; filter: grayscale(80%); }
.tegel-voltooid .titel { text-decoration: line-through; }

.weer-waarschuwing { position: absolute; top: -10px; left: -10px; font-size: 1.4rem; z-index: 5; animation: pulse 1s infinite alternate; }
@keyframes pulse { from { transform: scale(1); } to { transform: scale(1.2); } }

@keyframes pop-in { 0% { transform: scale(0.3); opacity: 0; } 70% { transform: scale(1.2); opacity: 1; } 100% { transform: scale(1); opacity: 1; } }
.vinkje { position: absolute; top: -8px; right: -8px; background-color: var(--oranje-vinkje); color: white; width: 20px; height: 20px; border-radius: 50%; font-size: 12px; display: flex; align-items: center; justify-content: center; border: 2px solid white; font-weight: bold; animation: pop-in 0.3s forwards; }

/* Bodempaneel & Knoppen */
.bottom-panel { position: absolute; bottom: 0; width: 100%; max-width: 400px; box-sizing: border-box; background-color: rgba(255, 255, 255, 0.98); padding: 15px 20px; text-align: center; border-top: 1px solid #eee; box-shadow: 0 -5px 15px rgba(0,0,0,0.05); z-index: 10; }
.bottom-panel h3 { margin: 0 0 10px 0; color: #333; font-weight: 500; font-size: 1rem; }
.bottom-panel p { margin: 0; color: var(--tekst-grijs); font-size: 0.85rem; }

.actie-knoppen-rij { display: flex; gap: 8px; }
.actie-knop { flex: 1; padding: 10px 5px; border: none; border-radius: 6px; font-size: 0.85rem; cursor: pointer; font-weight: bold; color: white; transition: all 0.2s ease; }
.actie-knop:active { transform: scale(0.95); opacity: 0.9; }

.afrond-knop { background-color: var(--groen-knop); }
.verplaats-knop { background-color: var(--blauw-knop); }
.verwijder-knop { background-color: var(--rood-knop); }

.nieuwe-actie-knop { width: 100%; padding: 12px; background-color: var(--teal-licht); color: white; border: none; border-radius: 6px; font-weight: bold; font-size: 1rem; cursor: pointer; }
.nieuwe-actie-knop:active { background-color: var(--teal-donker); }

/* Formulier */
.formulier { display: flex; flex-direction: column; gap: 10px; }
.form-rij { display: flex; gap: 10px; }
.form-input { padding: 10px; border: 1px solid #ccc; border-radius: 6px; font-family: inherit; font-size: 0.9rem; }
.form-tijd { width: 80px; }
.form-tekst { flex: 1; }

.kies-dag-lijst { display: flex; flex-direction: column; gap: 8px; margin-bottom: 10px; max-height: 150px; overflow-y: auto; }
.kies-dag-knop { padding: 10px; background-color: #f8f9fa; border: 1px solid #ddd; border-radius: 4px; font-size: 0.9rem; color: var(--teal-donker); font-weight: bold; cursor: pointer; transition: background-color 0.2s; }
.kies-dag-knop:active { background-color: #e9ecef; }
.annuleer-knop { width: 100%; padding: 10px; background: none; border: none; color: var(--tekst-grijs); text-decoration: underline; cursor: pointer; }

/* Gids & Animaties */
.hoofd-titel-gids { font-size: 1.4rem; color: var(--teal-donker); margin-bottom: 20px; text-align: center; }
.categorie-sectie { margin-bottom: 30px; }
.gids-categorie-titel { font-size: 1.1rem; color: #333; margin-bottom: 10px; padding-bottom: 5px; border-bottom: 2px solid var(--teal-licht); }

.gids-lijst { display: flex; flex-direction: column; gap: 15px; }
.gids-kaart { background-color: white; border-left: 5px solid var(--teal-donker); border-radius: 8px; padding: 15px; display: flex; align-items: center; box-shadow: 0 2px 8px rgba(0,0,0,0.05); transition: all 0.3s ease; }
.gids-icoon { font-size: 2.5rem; margin-right: 15px; background: #f4f8f6; padding: 10px; border-radius: 50%; }
.gids-info h3 { margin: 0 0 5px 0; color: var(--teal-donker); font-size: 1.1rem; }
.gids-detail { margin: 0 0 3px 0; color: var(--tekst-grijs); font-size: 0.85rem; }

.laad-scherm { text-align: center; padding: 40px 0; color: var(--tekst-grijs); }
.spinner { border: 4px solid rgba(0,0,0,0.1); width: 40px; height: 40px; border-radius: 50%; border-left-color: var(--teal-donker); animation: draai 1s linear infinite; margin: 0 auto 15px auto; }
@keyframes draai { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }

.fade-enter-active, .fade-leave-active { transition: opacity 0.25s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
.lijst-enter-active, .lijst-leave-active { transition: all 0.4s cubic-bezier(0.25, 0.8, 0.25, 1); }
.lijst-enter-from, .lijst-leave-to { opacity: 0; transform: scale(0.8) translateY(20px); }
.lijst-move { transition: transform 0.4s cubic-bezier(0.25, 0.8, 0.25, 1); }
</style>