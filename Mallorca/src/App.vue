<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue';

const huidigScherm = ref('home'); 
const toonVerplaatsMenu = ref(false);

// --- 1. AFTELKLOK LOGICA ---
const dagenTeGaan = ref(0);
const urenTeGaan = ref(0);
const minutenTeGaan = ref(0);
const secondenTeGaan = ref(0);
let aftelTimer: number;

const startCountdown = () => {
  const vertrekDatum = new Date('2026-07-08T08:30:00').getTime();
  const updateTimer = () => {
    const nu = new Date().getTime();
    const verschil = vertrekDatum - nu;
    
    if(verschil > 0) {
      dagenTeGaan.value = Math.floor(verschil / (1000 * 60 * 60 * 24));
      urenTeGaan.value = Math.floor((verschil % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
      minutenTeGaan.value = Math.floor((verschil % (1000 * 60 * 60)) / (1000 * 60));
      secondenTeGaan.value = Math.floor((verschil % (1000 * 60)) / 1000);
    }
  };
  
  updateTimer(); 
  aftelTimer = setInterval(updateTimer, 1000); 
};

// --- 2. LIVE WEER LOGICA ---
const weerCalaRatjada = ref({ temp: '--', conditie: 'Weer laden...', icoon: '⏳' });
const weerHaren = ref({ temp: '--', conditie: 'Weer laden...', icoon: '⏳' });

const haalWeerOp = async () => {
  // 👇 VUL HIERONDER JOUW OPENWEATHERMAP API SLEUTEL IN 👇
  const apiKey = '26a57f31f3ae779576631ba7e3ac4894'; 

  // Check of de sleutel al is ingevuld
  if (apiKey === '26a57f31f3ae779576631ba7e3ac4894') {
    weerCalaRatjada.value.conditie = 'Sleutel nog niet ingesteld';
    weerHaren.value.conditie = 'Sleutel nog niet ingesteld';
    return;
  }

  // Zet weercodes om in emoji's
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
    // Haal weer voor Cala Ratjada, Spanje
    const resMallorca = await fetch(`https://api.openweathermap.org/data/2.5/weather?q=Cala%20Ratjada,es&units=metric&lang=nl&appid=${apiKey}`);
    const dataMallorca = await resMallorca.json();
    if(dataMallorca.main) {
      weerCalaRatjada.value.temp = Math.round(dataMallorca.main.temp).toString();
      weerCalaRatjada.value.conditie = dataMallorca.weather[0].description;
      weerCalaRatjada.value.icoon = getIcoon(dataMallorca.weather[0].icon);
    }

    // Haal weer voor Haren, Nederland
    const resHaren = await fetch(`https://api.openweathermap.org/data/2.5/weather?q=Haren,nl&units=metric&lang=nl&appid=${apiKey}`);
    const dataHaren = await resHaren.json();
    if(dataHaren.main) {
      weerHaren.value.temp = Math.round(dataHaren.main.temp).toString();
      weerHaren.value.conditie = dataHaren.weather[0].description;
      weerHaren.value.icoon = getIcoon(dataHaren.weather[0].icon);
    }
  } catch (error) {
    console.error("Fout bij ophalen weer:", error);
    weerCalaRatjada.value.conditie = 'Fout bij ophalen';
    weerHaren.value.conditie = 'Fout bij ophalen';
  }
};

// --- START FUNCTIES ---
onMounted(() => {
  startCountdown();
  haalWeerOp(); // Start de weer-api
});

onUnmounted(() => {
  clearInterval(aftelTimer);
});

// --- 3. DAGSCHEMA LOGICA ---
const dagschema = ref([
  {
    id: 1, datum: 'Woensdag 8 juli 2026', titel: 'Dag 1: Reisdag',
    activiteiten: [
      { id: 101, tijd: '08:30', omschrijving: 'Vertrek auto Haren', icoon: '🚗', geselecteerd: false },
      { id: 102, tijd: '10:30', omschrijving: 'Aankomst FMO', icoon: '🅿️', geselecteerd: false },
      { id: 103, tijd: '12:50', omschrijving: 'Vlucht FR7386', icoon: '✈️', geselecteerd: false },
      { id: 104, tijd: '15:10', omschrijving: 'Aankomst Palma', icoon: '🛬', geselecteerd: false },
      { id: 105, tijd: '16:00', omschrijving: 'Huurauto ophalen', icoon: '🚙', geselecteerd: false },
      { id: 106, tijd: '17:15', omschrijving: 'Inchecken hotel', icoon: '🏨', geselecteerd: false },
      { id: 107, tijd: '19:30', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false }
    ]
  },
  {
    id: 2, datum: 'Donderdag 9 juli 2026', titel: 'Dag 2: Rust & strand',
    activiteiten: [
      { id: 201, tijd: '09:30', omschrijving: 'Rustig ontbijten', icoon: '☕', geselecteerd: false },
      { id: 202, tijd: '11:00', omschrijving: 'Chillen zwembad', icoon: '🌊', geselecteerd: false },
      { id: 203, tijd: '13:30', omschrijving: 'Snelle lunch', icoon: '🍴', geselecteerd: false },
      { id: 204, tijd: '14:30', omschrijving: 'Duik Cala Agulla', icoon: '🏖️', geselecteerd: false },
      { id: 205, tijd: '19:00', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false }
    ]
  },
  {
    id: 3, datum: 'Vrijdag 10 juli 2026', titel: 'Dag 3: Kust & snorkelen',
    activiteiten: [
      { id: 301, tijd: '09:00', omschrijving: 'Gezamenlijk ontbijt', icoon: '☕', geselecteerd: false },
      { id: 302, tijd: '10:30', omschrijving: 'Rit Cala Mesquida', icoon: '🚙', geselecteerd: false },
      { id: 303, tijd: '11:00', omschrijving: 'Wandeling kustpad', icoon: '🚶', geselecteerd: false },
      { id: 304, tijd: '12:00', omschrijving: 'Snorkelen baai', icoon: '🐟', geselecteerd: false },
      { id: 305, tijd: '14:00', omschrijving: 'Lichte lunch', icoon: '🥗', geselecteerd: false },
      { id: 306, tijd: '15:30', omschrijving: 'Zwembad', icoon: '🌊', geselecteerd: false },
      { id: 307, tijd: '19:30', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false }
    ]
  },
  {
    id: 4, datum: 'Zaterdag 11 juli 2026', titel: 'Dag 4: Fietsen & afkoelen',
    activiteiten: [
      { id: 401, tijd: '08:00', omschrijving: 'Racefiets ophalen (Ewout)', icoon: '🚲', geselecteerd: false },
      { id: 402, tijd: '08:30', omschrijving: 'Bergrit Artà (Ewout)', icoon: '⛰️', geselecteerd: false },
      { id: 403, tijd: '08:30', omschrijving: 'Uitslapen (Vera & Tris)', icoon: '😴', geselecteerd: false },
      { id: 404, tijd: '11:30', omschrijving: 'Fiets inleveren (Ewout)', icoon: '🚲', geselecteerd: false },
      { id: 405, tijd: '13:30', omschrijving: 'Lunch', icoon: '🍴', geselecteerd: false },
      { id: 406, tijd: '15:00', omschrijving: 'Cala Gat', icoon: '🏖️', geselecteerd: false },
      { id: 407, tijd: '19:00', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false }
    ]
  },
  {
    id: 5, datum: 'Zondag 12 juli 2026', titel: 'Dag 5: Beachclub',
    activiteiten: [
      { id: 501, tijd: '09:30', omschrijving: 'Ontbijt', icoon: '☕', geselecteerd: false },
      { id: 502, tijd: '11:00', omschrijving: 'Rit strandclub', icoon: '🍹', geselecteerd: false },
      { id: 503, tijd: '13:00', omschrijving: 'Tapas aan zee', icoon: '🥘', geselecteerd: false },
      { id: 504, tijd: '15:30', omschrijving: 'Ontspanning hotel', icoon: '😎', geselecteerd: false },
      { id: 505, tijd: '19:30', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false }
    ]
  },
  {
    id: 6, datum: 'Maandag 13 juli 2026', titel: 'Dag 6: Winkelen',
    activiteiten: [
      { id: 601, tijd: '09:00', omschrijving: 'Ontbijten', icoon: '☕', geselecteerd: false },
      { id: 602, tijd: '10:00', omschrijving: 'Winkelen stad', icoon: '🛍️', geselecteerd: false },
      { id: 603, tijd: '14:00', omschrijving: 'Late lunch', icoon: '🌮', geselecteerd: false },
      { id: 604, tijd: '15:30', omschrijving: 'Terugrit', icoon: '🚙', geselecteerd: false },
      { id: 605, tijd: '17:00', omschrijving: 'Zwembad', icoon: '🌊', geselecteerd: false },
      { id: 606, tijd: '19:30', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false }
    ]
  },
  {
    id: 7, datum: 'Dinsdag 14 juli 2026', titel: 'Dag 7: Cala Lliteras',
    activiteiten: [
      { id: 701, tijd: '09:30', omschrijving: 'Ontbijt', icoon: '☕', geselecteerd: false },
      { id: 702, tijd: '11:00', omschrijving: 'Snorkelen Lliteras', icoon: '🐟', geselecteerd: false },
      { id: 703, tijd: '13:30', omschrijving: 'Lichte lunch', icoon: '🍴', geselecteerd: false },
      { id: 704, tijd: '14:30', omschrijving: 'Lezen bij hotel', icoon: '📖', geselecteerd: false },
      { id: 705, tijd: '19:00', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false }
    ]
  },
  {
    id: 8, datum: 'Woensdag 15 juli 2026', titel: 'Dag 8: Vuurtoren',
    activiteiten: [
      { id: 801, tijd: '09:30', omschrijving: 'Ontbijt', icoon: '☕', geselecteerd: false },
      { id: 802, tijd: '11:00', omschrijving: 'Vuurtoren wandeling', icoon: '🗼', geselecteerd: false },
      { id: 803, tijd: '13:00', omschrijving: 'Lunch', icoon: '🥗', geselecteerd: false },
      { id: 804, tijd: '14:00', omschrijving: 'Zwemmen', icoon: '🌊', geselecteerd: false },
      { id: 805, tijd: '19:30', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false }
    ]
  },
  {
    id: 9, datum: 'Donderdag 16 juli 2026', titel: 'Dag 9: Laatste duik',
    activiteiten: [
      { id: 901, tijd: '09:30', omschrijving: 'Ontbijt', icoon: '☕', geselecteerd: false },
      { id: 902, tijd: '11:00', omschrijving: 'Ontspanning zwembad', icoon: '😎', geselecteerd: false },
      { id: 903, tijd: '13:30', omschrijving: 'Lunch', icoon: '🍴', geselecteerd: false },
      { id: 904, tijd: '14:30', omschrijving: 'Souvenirs scoren', icoon: '🛍️', geselecteerd: false },
      { id: 905, tijd: '19:00', omschrijving: 'Diner in hotel', icoon: '🍽️', geselecteerd: false }
    ]
  },
  {
    id: 10, datum: 'Vrijdag 17 juli 2026', titel: 'Dag 10: Terugreis',
    activiteiten: [
      { id: 1001, tijd: '09:00', omschrijving: 'Koffers sluiten', icoon: '🎒', geselecteerd: false },
      { id: 1002, tijd: '10:00', omschrijving: 'Uitchecken', icoon: '🏨', geselecteerd: false },
      { id: 1003, tijd: '10:30', omschrijving: 'Rit vliegveld', icoon: '🚙', geselecteerd: false },
      { id: 1004, tijd: '11:45', omschrijving: 'Auto inleveren', icoon: '🔑', geselecteerd: false },
      { id: 1005, tijd: '13:45', omschrijving: 'Vlucht FR7282', icoon: '✈️', geselecteerd: false },
      { id: 1006, tijd: '16:25', omschrijving: 'Aankomst FMO', icoon: '🛬', geselecteerd: false },
      { id: 1007, tijd: '17:00', omschrijving: 'Rit naar huis', icoon: '🚗', geselecteerd: false },
      { id: 1008, tijd: '18:45', omschrijving: 'Aankomst Haren', icoon: '🏠', geselecteerd: false }
    ]
  }
]);

const toggleSelectie = (activiteit: any) => {
  activiteit.geselecteerd = !activiteit.geselecteerd;
};

const heeftSelectie = computed(() => {
  return dagschema.value.some(dag => dag.activiteiten.some(act => act.geselecteerd));
});

const verwijderGeselecteerde = () => {
  dagschema.value.forEach(dag => {
    dag.activiteiten = dag.activiteiten.filter(act => !act.geselecteerd);
  });
  toonVerplaatsMenu.value = false; 
};

const verplaatsGeselecteerdeNaar = (doelDagId: number) => {
  let teVerplaatsen: any[] = [];
  dagschema.value.forEach(dag => {
    const geselecteerd = dag.activiteiten.filter(act => act.geselecteerd);
    teVerplaatsen.push(...geselecteerd);
    dag.activiteiten = dag.activiteiten.filter(act => !act.geselecteerd);
  });

  const doelDag = dagschema.value.find(dag => dag.id === doelDagId);
  if (doelDag) {
    teVerplaatsen.forEach(act => {
      act.geselecteerd = false; 
      doelDag.activiteiten.push(act);
    });
    doelDag.activiteiten.sort((a, b) => a.tijd.localeCompare(b.tijd));
  }
  toonVerplaatsMenu.value = false;
};
</script>

<template>
  <main class="app-container">
    <header class="app-header">
      <h1>Mallorca 2026</h1>
    </header>

    <nav class="hoofd-menu">
      <button :class="{ 'menu-actief': huidigScherm === 'home' }" @click="huidigScherm = 'home'">🏠 Home</button>
      <button :class="{ 'menu-actief': huidigScherm === 'planner' }" @click="huidigScherm = 'planner'">📅 Planner</button>
      <button :class="{ 'menu-actief': huidigScherm === 'gids' }" @click="huidigScherm = 'gids'">🏖️ Gids</button>
    </nav>

    <div class="content-gebied">
      <transition name="fade" mode="out-in">
        
        <div v-if="huidigScherm === 'home'" class="home-scherm">
          <h2>Welkom Weiner dogs!</h2>
          
          <div class="weer-kaart countdown-kaart">
            <h3>⏳ Aftellen naar vertrek</h3>
            <div class="weer-info">{{ dagenTeGaan }}d {{ urenTeGaan }}u {{ minutenTeGaan }}m <span class="seconden">{{ secondenTeGaan }}s</span></div>
            <p>tot woensdag 8 juli 2026!</p>
          </div>

          <div class="weer-dashboard">
            <div class="weer-kaart">
              <h3>📍 Cala Ratjada</h3>
              <div class="weer-info">{{ weerCalaRatjada.icoon }} {{ weerCalaRatjada.temp }}°C</div>
              <p class="weer-beschrijving">{{ weerCalaRatjada.conditie }}</p>
            </div>
            <div class="weer-kaart">
              <h3>📍 Haren (Gn)</h3>
              <div class="weer-info">{{ weerHaren.icoon }} {{ weerHaren.temp }}°C</div>
              <p class="weer-beschrijving">{{ weerHaren.conditie }}</p>
            </div>
          </div>
        </div>

        <div v-else-if="huidigScherm === 'planner'">
          <div v-for="dag in dagschema" :key="dag.id" class="dag-sectie">
            <h2 class="dag-titel">{{ dag.titel }}</h2>
            
            <transition-group name="lijst" tag="div" class="activiteiten-grid">
              <div 
                v-for="activiteit in dag.activiteiten" 
                :key="activiteit.id" 
                class="tegel"
                :class="{ 'tegel-actief': activiteit.geselecteerd }"
                @click="toggleSelectie(activiteit)"
              >
                <div v-if="activiteit.geselecteerd" class="vinkje">✓</div>
                <div class="icoon">{{ activiteit.icoon }}</div>
                <div class="titel">{{ activiteit.omschrijving }}</div>
                <div class="tijd">{{ activiteit.tijd }}</div>
              </div>
            </transition-group>
            
          </div>
        </div>

        <div v-else-if="huidigScherm === 'gids'">
          <h2>Strand & snorkelgids</h2>
          <p>Hier komt straks de interactieve lijst met baaitjes.</p>
        </div>

      </transition>
    </div>
    
    <div class="bottom-panel" v-if="huidigScherm === 'planner'">
      <transition name="fade" mode="out-in">
        <div v-if="heeftSelectie" key="acties">
          <div v-if="!toonVerplaatsMenu">
            <h3>Geselecteerde acties</h3>
            <div class="actie-knoppen-rij">
              <button class="actie-knop verplaats-knop" @click="toonVerplaatsMenu = true">
                ➡️ Verplaatsen
              </button>
              <button class="actie-knop verwijder-knop" @click="verwijderGeselecteerde">
                🗑️ Verwijderen
              </button>
            </div>
          </div>

          <div v-else>
            <h3>Verplaatsen naar:</h3>
            <div class="kies-dag-lijst">
              <button 
                v-for="dag in dagschema" 
                :key="dag.id" 
                class="kies-dag-knop"
                @click="verplaatsGeselecteerdeNaar(dag.id)"
              >
                {{ dag.titel }}
              </button>
            </div>
            <button class="annuleer-knop" @click="toonVerplaatsMenu = false">✖ Annuleren</button>
          </div>

        </div>
        <div v-else key="leeg">
          <h3>Wat ga je er doen?</h3>
          <p>Selecteer de activiteiten om ze te bewerken of te verplaatsen naar een andere dag.</p>
        </div>
      </transition>
    </div>
  </main>
</template>

<style>
/* CSS Variabelen */
:root {
  --teal-licht: #a0ceb9;
  --teal-donker: #7bb29e;
  --oranje-vinkje: #f39c12;
  --blauw-knop: #3498db;
  --rood-knop: #e74c3c;
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
}

.app-container {
  width: 100%;
  max-width: 400px;
  background-color: var(--achtergrond);
  min-height: 100vh;
  position: relative;
  box-shadow: 0 0 20px rgba(0,0,0,0.1);
  overflow-x: hidden;
}

.app-header {
  background-color: var(--teal-licht);
  color: white;
  padding: 15px 20px;
  text-align: center;
}

.app-header h1 {
  margin: 0;
  font-size: 1.4rem;
  font-weight: 600;
}

.hoofd-menu {
  display: flex;
  background-color: white;
  border-bottom: 2px solid #eee;
}

.hoofd-menu button {
  flex: 1;
  padding: 12px 0;
  border: none;
  background: none;
  font-size: 0.9rem;
  color: var(--tekst-grijs);
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.2s;
}

.hoofd-menu button.menu-actief {
  color: var(--teal-donker);
  border-bottom: 3px solid var(--teal-donker);
}

.weer-dashboard {
  display: flex;
  flex-direction: column;
  gap: 15px;
  margin-top: 20px;
}

.weer-kaart {
  background-color: white;
  border: 2px solid var(--teal-licht);
  border-radius: 8px;
  padding: 20px;
  text-align: center;
  transition: transform 0.2s ease;
}

.countdown-kaart {
  background-color: var(--teal-licht);
  color: white;
  margin-top: 20px;
  border-color: var(--teal-donker);
}

.countdown-kaart h3 { color: white; margin-bottom: 10px; font-size: 1.1rem; }
.countdown-kaart .weer-info { color: white; font-size: 1.8rem; letter-spacing: 1px; }
.countdown-kaart .seconden { font-size: 1.4rem; opacity: 0.8; }
.countdown-kaart p { color: white; opacity: 0.9; margin-top: 10px;}

.weer-kaart h3 { margin: 0 0 10px 0; color: var(--teal-donker); }
.weer-info { font-size: 2.5rem; font-weight: bold; color: #333; }
.weer-kaart p { color: var(--tekst-grijs); margin: 5px 0 0 0; font-size: 0.8rem; }
.weer-beschrijving { text-transform: capitalize; }

.content-gebied {
  padding: 20px 15px;
  padding-bottom: 150px; 
}

.dag-titel {
  font-size: 1.1rem;
  color: var(--teal-donker);
  margin-bottom: 15px;
  border-bottom: 1px solid #eee;
  padding-bottom: 5px;
}

.activiteiten-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
  margin-bottom: 30px;
  position: relative;
}

.tegel {
  background-color: white;
  border: 2px solid var(--teal-licht);
  border-radius: 4px;
  padding: 15px 5px;
  text-align: center;
  cursor: pointer;
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 90px;
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
}

.tegel:active { transform: scale(0.92); }

.icoon { font-size: 2rem; margin-bottom: 8px; }
.titel { color: var(--teal-donker); font-size: 0.8rem; font-weight: 600; line-height: 1.2; }
.tijd { color: var(--tekst-grijs); font-size: 0.7rem; margin-top: 4px; }

.tegel-actief {
  background-color: var(--teal-donker);
  border-color: var(--teal-donker);
}
.tegel-actief .titel, .tegel-actief .tijd { color: white; }

@keyframes pop-in {
  0% { transform: scale(0.3); opacity: 0; }
  70% { transform: scale(1.2); opacity: 1; }
  100% { transform: scale(1); opacity: 1; }
}

.vinkje {
  position: absolute;
  top: -8px;
  right: -8px;
  background-color: var(--oranje-vinkje);
  color: white;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  font-size: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 2px solid white;
  font-weight: bold;
  animation: pop-in 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards;
}

.bottom-panel {
  position: sticky; 
  bottom: 0;
  width: 100%;
  box-sizing: border-box;
  background-color: rgba(255, 255, 255, 0.98);
  padding: 15px 20px;
  text-align: center;
  border-top: 1px solid #eee;
  box-shadow: 0 -5px 15px rgba(0,0,0,0.05); 
  z-index: 10;
}

.bottom-panel h3 { margin: 0 0 10px 0; color: #333; font-weight: 500; font-size: 1rem; }
.bottom-panel p { margin: 0; color: var(--tekst-grijs); font-size: 0.85rem; }

.actie-knoppen-rij { display: flex; gap: 10px; }

.actie-knop {
  flex: 1; 
  padding: 12px 10px;
  border: none;
  border-radius: 6px;
  font-size: 0.9rem;
  cursor: pointer;
  font-weight: bold;
  color: white;
  transition: all 0.2s ease;
}
.actie-knop:active { transform: scale(0.95); opacity: 0.9; }

.verplaats-knop { background-color: var(--blauw-knop); }
.verwijder-knop { background-color: var(--rood-knop); }

.kies-dag-lijst {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 10px;
  max-height: 150px;
  overflow-y: auto; 
}
.kies-dag-knop {
  padding: 10px;
  background-color: #f8f9fa;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 0.9rem;
  color: var(--teal-donker);
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.2s;
}
.kies-dag-knop:active { background-color: #e9ecef; }
.annuleer-knop {
  width: 100%;
  padding: 10px;
  background: none;
  border: none;
  color: var(--tekst-grijs);
  text-decoration: underline;
  cursor: pointer;
}

/* Tabbladen fade-overgang */
.fade-enter-active, .fade-leave-active { transition: opacity 0.25s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

/* Activiteitenlijst animaties */
.lijst-enter-active, .lijst-leave-active { transition: all 0.4s cubic-bezier(0.25, 0.8, 0.25, 1); }
.lijst-enter-from, .lijst-leave-to { opacity: 0; transform: scale(0.8) translateY(20px); }
.lijst-move { transition: transform 0.4s cubic-bezier(0.25, 0.8, 0.25, 1); }
</style>