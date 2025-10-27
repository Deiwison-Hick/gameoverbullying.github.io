  / ======================

const GAME_CONFIG = {
  eixo: "Bullying Escolar: Cultura de Paz",
  titulo: "A Escolha Está em Suas Mãos",
  introducao: "Você está no corredor da escola quando presencia uma situação de bullying. Suas escolhas vão definir o desfecho.",
  // Estrutura de cenas estilo árvore. Cada nó tem: id, texto, escolhas[] ou final{tipo, mensagem}
  cenas: [
    {
      id: "inicio",
      texto: "Você vê dois alunos cercando seu colega e rindo dele. Ele olha para você em silêncio. O que você faz?",
      escolhas: [
        { label: "Ignorar e seguir para a sala", goto: "ignorar" },
        { label: "Intervir e apoiar o colega", goto: "intervir" },
        { label: "Gravar com o celular e postar", goto: "gravar" }
      ]
    },
    {
      id: "ignorar",
      texto: "Você passa reto. A cena continua. No fundo, algo te incomoda.",
      escolhas: [
        { label: "Voltar e procurar um professor", goto: "professor" },
        { label: "Continuar fingindo que nada aconteceu", goto: "fim_ruim" }
      ]
    },
    {
      id: "intervir",
      texto: "Você pede para que parem. O clima pesa, mas seu colega respira aliviado.",
      escolhas: [
        { label: "Chamar um adulto responsável", goto: "professor" },
        { label: "Enfrentar os agressores sozinho", goto: "fim_neutro" }
      ]
    },
    {
      id: "gravar",
      texto: "Você grava a cena. O vídeo viraliza e a situação piora.",
      escolhas: [
        { label: "Apagar o vídeo e pedir desculpas", goto: "intervir" },
        { label: "Deixar o vídeo circular", goto: "fim_ruim" }
      ]
    },
    {
      id: "professor",
      texto: "Um professor chega, interrompe a agressão e acolhe as vítimas. Vocês combinam uma roda de conversa.",
      final: { tipo: "bom", mensagem: "Fim bom — Você foi um aliado. Empatia + Ação = Cultura de Paz." }
    },
    {
      id: "fim_neutro",
      texto: "Você arriscou sozinho. Não houve agressão física, mas foi tenso. Faltou buscar apoio.",
      final: { tipo: "neutro", mensagem: "Fim neutro — Coragem não precisa ser solitária." }
    },
    {
      id: "fim_ruim",
      texto: "O silêncio reforça a violência. Seu colega se afasta e perde o interesse pela escola.",
      final: { tipo: "ruim", mensagem: "Fim ruim — Quando ninguém age, o bullying vence." }
    }
  ],
  // Mensagens de badges
  badges: {
    bom: "Badge • Aliado da Cultura de Paz",
    neutro: "Badge • Você tem coragem — busque aliados",
    ruim: "Badge • Aprendizado: omissão também é escolha"
  }
};

// ======================
// Estado e Persistência
// ======================

const STORAGE_KEY = "feculema_game_state_v1";
let state = {
  visited: new Set(),
  finais: { bom: 0, neutro: 0, ruim: 0 },
  jogadores: 0,
  currentId: "inicio"
};

function saveState(){
  const toSave = {
    visited: Array.from(state.visited),
    finais: state.finais,
    jogadores: state.jogadores
  };
  localStorage.setItem(STORAGE_KEY, JSON.stringify(toSave));
}

function loadState(){
  try{
    const raw = localStorage.getItem(STORAGE_KEY);
    if(!raw) return;
    const parsed = JSON.parse(raw);
    state.visited = new Set(parsed.visited || []);
    state.finais = parsed.finais || { bom: 0, neutro: 0, ruim: 0 };
    state.jogadores = parsed.jogadores || 0;
  }catch(e){}
}

// ======================
// Motor do jogo
// ======================

const viewport = document.getElementById("viewport");
const progressBar = document.getElementById("progressBar");
const metricas = document.getElementById("metricas");
const btnReiniciar = document.getElementById("btnReiniciar");
const btnCompartilhar = document.getElementById("btnCompartilhar");

function getCena(id){ return GAME_CONFIG.cenas.find(c => c.id === id); }

function renderCena(id){
  state.currentId = id;
  state.visited.add(id);
  saveState();
  updateProgress();
  const cena = getCena(id);
  if(!cena){ viewport.innerHTML = "<p>Erro: cena não encontrada.</p>"; return; }

  if(cena.final){
    const tipo = cena.final.tipo;
    state.finais[tipo]++;
    saveState();
    updateMetricas();
    viewport.innerHTML = `
      <div class="card">
        <p class="meta">${GAME_CONFIG.eixo}</p>
        <h3>${GAME_CONFIG.titulo}</h3>
        <p>${cena.texto}</p>
        <p class="badge">${GAME_CONFIG.badges[tipo]}</p>
        <p><strong>${cena.final.mensagem}</strong></p>
        <div class="choice-list">
          <a class="choice" href="#" data-goto="inicio">Jogar de novo</a>
          <a class="choice" href="#" data-share="true">Compartilhar resultado</a>
        </div>
      </div>
    `;
  }else{
    viewport.innerHTML = `
      <div class="card">
        <p class="meta">${GAME_CONFIG.eixo}</p>
        <h3>${GAME_CONFIG.titulo}</h3>
        <p>${cena.texto}</p>
        <div class="choice-list">
          ${cena.escolhas.map((e,i)=>`<a class="choice" href="#" data-goto="${e.goto}">${e.label}</a>`).join("")}
        </div>
      </div>
    `;
  }

  viewport.querySelectorAll("[data-goto]").forEach(el=>{
    el.addEventListener("click", (ev)=>{
      ev.preventDefault();
      renderCena(el.getAttribute("data-goto"));
    });
  });

  viewport.querySelectorAll("[data-share]").forEach(el=>{
    el.addEventListener("click", async (ev)=>{
      ev.preventDefault();
      await compartilharResultado();
    });
  });
}

function updateProgress(){
  // progresso = % de nós visitados (não perfeito para árvores, mas dá noção de avanço)
  const total = GAME_CONFIG.cenas.length;
  const visited = state.visited.size;
  const pct = Math.min(100, Math.round((visited/total)*100));
  progressBar.style.width = pct + "%";
  progressBar.setAttribute("aria-valuenow", pct);
}

function updateMetricas(){
  metricas.textContent = `${state.jogadores} jogadores • ${state.finais.bom} finais bons • ${state.finais.neutro} finais neutros • ${state.finais.ruim} finais ruins`;
}

async function compartilharResultado(){
  const cena = getCena(state.currentId);
  const tipo = cena?.final?.tipo || "progresso";
  const msg = (cena?.final?.mensagem) ? `Resultado: ${cena.final.mensagem}` : "Estou jogando e aprendendo sobre cultura de paz!";
  const shareData = {
    title: "FECULEMA • Jogo Interativo",
    text: `${msg} — Jogue você também!`,
    url: location.href
  };
  try{
    if(navigator.share){
      await navigator.share(shareData);
    }else{
      await navigator.clipboard.writeText(`${shareData.text} ${shareData.url}`);
      alert("Link copiado para a área de transferência!");
    }
  }catch(e){
    console.log("Compartilhar cancelado/erro:", e);
  }
}

// ======================
// Controles
// ======================
btnReiniciar.addEventListener("click", ()=>{
  state.visited = new Set();
  state.currentId = "inicio";
  state.jogadores += 1;
  saveState();
  updateMetricas();
  renderCena("inicio");
});

btnCompartilhar.addEventListener("click", (e)=>{
  e.preventDefault();
  compartilharResultado();
});

// ======================
// Boot
// ======================
(function init(){
  loadState();
  // Conta um jogador quando carrega pela primeira vez
  if(!localStorage.getItem("feculema_game_first_visit")){
    state.jogadores += 1;
    localStorage.setItem("feculema_game_first_visit","1");
  }
  updateMetricas();
  renderCena("inicio");
})();    coloque ai 