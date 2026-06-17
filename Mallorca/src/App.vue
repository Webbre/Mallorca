<script setup lang="ts">
import { ref, computed } from 'vue';

const huidigScherm = ref('home'); 

const dagschema = ref([
  {
    id: 1,
    datum: 'Woensdag 8 juli 2026',
    titel: 'Dag 1: Reisdag',
    activiteiten: [
      { id: 101, tijd: '08:30', omschrijving: 'Vertrek', icoon: '🚗', geselecteerd: false },
      { id: 102, tijd: '12:50', omschrijving: 'Vlucht', icoon: '✈️', geselecteerd: false },
      { id: 103, tijd: '17:15', omschrijving: 'Inchecken', icoon: '🏨', geselecteerd: false }
    ]
  },
  {
    id: 2,
    datum: 'Donderdag 9 juli 2026',
    titel: 'Dag 2: Rust & Strand',
    activiteiten: [
      { id: 201, tijd: '09:30', omschrijving: 'Ontbijt', icoon: '🥐', geselecteerd: false },
      { id: 202, tijd: '11:00', omschrijving: 'Zwembad', icoon: '🏊‍♂️', geselecteerd: false },
      { id: 203, tijd: '14:30', omschrijving: 'Strand', icoon: '🏖️', geselecteerd: false },
      { id: 204, tijd: '19:00', omschrijving: 'Diner', icoon: '🍽️', geselecteerd: false }
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
};
</script>

<template>
  <main class="app-container">
    <header class="app-header">
      <h1>Mallorca 2026</h1>
    </header>

    <nav class="hoofd-menu">
      <button 
        :class="{ 'menu-actief': huidigScherm === 'home' }" 
        @click="huidigScherm = 'home'"
      >
        🏠 Home
      </button>
      <button 
        :class="{ 'menu-actief': huidigScherm === 'planner' }" 
        @click="huidigScherm = 'planner'"
      >
        📅 Planner
      </button>
      <button 
        :class="{ 'menu-actief': huidigScherm === 'gids' }" 
        @click="huidigScherm = 'gids'"
      >
        🏖️ Gids
      </button>
    </nav>

    <div class="content-gebied">
      
      <div v-if="huidigScherm === 'home'" class="home-scherm">
        <h2>Welkom Weiner dogs!</h2>
        <div class="weer-dashboard">
          <div class="weer-kaart">
            <h3>📍 Cala Ratjada</h3>
            <div class="weer-info">☀️ 28°C</div>
            <p>Actueel weer (Voorbeeld)</p>
          </div>
          <div class="weer-kaart">
            <h3>📍 Haren (Gn)</h3>
            <div class="weer-info">☁️ 18°C</div>
            <p>Actueel weer (Voorbeeld)</p>
          </div>
        </div>
      </div>

      <div v-if="huidigScherm === 'planner'">
        <div v-for="dag in dagschema" :key="dag.id" class="dag-sectie">
          <h2 class="dag-titel">{{ dag.titel }}</h2>
          <div class="activiteiten-grid">
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
          </div>
        </div>
      </div>

      <div v-if="huidigScherm === 'gids'">
        <h2>Strand & Snorkelgids</h2>
        <p>Hier komt straks de interactieve lijst met baaitjes.</p>
      </div>

    </div>
    
    <div class="bottom-panel" v-if="huidigScherm === 'planner'">
      <div v-if="heeftSelectie">
        <h3>Geselecteerde acties</h3>
        <button class="actie-knop verwijder-knop" @click="verwijderGeselecteerde">
          🗑️ Verwijderen
        </button>
      </div>
      <div v-else>
        <h3>Wat ga je er doen?</h3>
        <p>Selecteer de activiteiten om ze te bewerken of te verplaatsen naar een andere dag.</p>
      </div>
    </div>
  </main>
</template>

<style>
:root {
  --teal-licht: #a0ceb9;
  --teal-donker: #7bb29e;
  --oranje-vinkje: #f39c12;
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
}

.weer-kaart h3 {
  margin: 0 0 10px 0;
  color: var(--teal-donker);
}

.weer-info {
  font-size: 2.5rem;
  font-weight: bold;
  color: #333;
}

.weer-kaart p {
  color: var(--tekst-grijs);
  margin: 5px 0 0 0;
  font-size: 0.8rem;
}

.content-gebied {
  padding: 20px 15px;
  padding-bottom: 120px; 
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
}

.tegel {
  background-color: white;
  border: 2px solid var(--teal-licht);
  border-radius: 4px;
  padding: 15px 5px;
  text-align: center;
  cursor: pointer;
  position: relative;
  transition: all 0.2s ease;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 90px;
}

.icoon {
  font-size: 2rem;
  margin-bottom: 8px;
}

.titel {
  color: var(--teal-donker);
  font-size: 0.8rem;
  font-weight: 600;
}

.tijd {
  color: var(--tekst-grijs);
  font-size: 0.7rem;
  margin-top: 4px;
}

.tegel-actief {
  background-color: var(--teal-donker);
  border-color: var(--teal-donker);
}

.tegel-actief .titel, 
.tegel-actief .tijd {
  color: white;
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
}

/* AANGEPAST: Flexbox trucs om het paneel te centreren */
.bottom-panel {
  position: fixed;
  bottom: 0;
  left: 50%; 
  transform: translateX(-50%); 
  width: 100%;
  max-width: 400px;
  box-sizing: border-box;
  background-color: rgba(255, 255, 255, 0.95);
  padding: 20px;
  text-align: center;
  border-top: 1px solid #eee;
  z-index: 10;
}

.bottom-panel h3 {
  margin: 0 0 10px 0;
  color: #333;
  font-weight: 500;
}

.bottom-panel p {
  margin: 0;
  color: var(--tekst-grijs);
  font-size: 0.9rem;
}

.actie-knop {
  padding: 12px 20px;
  border: none;
  border-radius: 6px;
  font-size: 1rem;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.2s;
  width: 100%;
}

.verwijder-knop {
  background-color: #e74c3c;
  color: white;
}

.verwijder-knop:hover {
  background-color: #c0392b;
}
</style>