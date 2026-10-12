[controle-sementes (1).html](https://github.com/user-attachments/files/33359959/controle-sementes.1.html)
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="theme-color" content="#0e5a2d">
<title>Controle de Sementes Comigo</title>
<style>
:root{--verde:#0e5a2d;--verde2:#e8f3ec;--txt:#1d2b22;--borda:#cfdcd3;--erro:#b3261e;--aviso:#8a5a00}
*{box-sizing:border-box}
body{margin:0;font-family:system-ui,Arial,sans-serif;background:#f4f7f5;color:var(--txt)}
header{background:var(--verde);color:#fff;padding:16px;display:flex;justify-content:space-between;align-items:center;gap:8px}
header h1{font-size:18px;margin:0}
main{max-width:480px;margin:0 auto;padding:16px}
.card{background:#fff;border:1px solid var(--borda);border-radius:14px;padding:16px;margin-bottom:16px}
label{display:block;font-size:13px;font-weight:600;margin:12px 0 4px}
input{width:100%;padding:12px;font-size:16px;border:1px solid var(--borda);border-radius:10px;background:#fff;color:var(--txt)}
input:focus-visible,button:focus-visible{outline:3px solid #7fc79a;outline-offset:2px}
button{width:100%;margin-top:16px;padding:14px;font-size:16px;font-weight:700;color:#fff;background:var(--verde);border:0;border-radius:10px;cursor:pointer}
button:disabled{opacity:.6;cursor:wait}
button.sair{width:auto;margin:0;padding:8px 12px;font-size:13px;background:#ffffff33}
.msg{margin-top:12px;font-size:14px;min-height:18px}
.ok{color:var(--verde)}.erro{color:var(--erro)}
.topo{display:flex;justify-content:space-between;align-items:center;font-size:14px}
.pill{display:none;font-size:12px;font-weight:700;background:#fff6d6;color:var(--aviso);border:1px solid #ecd888;border-radius:999px;padding:4px 10px}
.pill.on{display:inline-block}
table{width:100%;border-collapse:collapse;font-size:13px;margin-top:8px}
th,td{text-align:left;padding:8px 4px;border-bottom:1px solid var(--borda)}
td.pend{color:var(--aviso);font-weight:600}
.vazio{font-size:13px;color:#5b6b61;margin:10px 0 0}
.demo{background:#fff6d6;border:1px solid #ecd888;border-radius:10px;padding:8px 12px;font-size:12px;margin-bottom:12px}
.hide{display:none}
</style>
</head>
<body>
<header>
  <h1>🌱 Controle de Sementes Comigo</h1>
  <button class="sair hide" id="btnSair" type="button">Sair</button>
</header>
<main>
  <div class="demo hide" id="demo">Modo demonstração: nada vai para a planilha ainda. Teste com usuário <b>maria</b> e senha <b>1234</b>.</div>

  <section class="card" id="telaLogin">
    <label for="usuario">Usuário</label>
    <input id="usuario" autocapitalize="none" autocomplete="username">
    <label for="senha">Senha</label>
    <input id="senha" type="password" autocomplete="current-password">
    <button id="btnEntrar" type="button">Entrar</button>
    <div class="msg" id="msgLogin" role="status" aria-live="polite"></div>
  </section>

  <div class="hide" id="telaLanc">
    <section class="card">
      <div class="topo">
        <span>Olá, <b id="nomeUser"></b></span>
        <span class="pill" id="pill"></span>
      </div>
      <label for="semente">Nome da semente</label>
      <input id="semente" autocomplete="off" autocapitalize="words" maxlength="80">
      <label for="ticket">Nº Ticket</label>
      <input id="ticket" inputmode="numeric" autocomplete="off">
      <label for="qtd">QTD (unidades)</label>
      <input id="qtd" type="number" inputmode="numeric" step="1" min="1">
      <button id="btnLancar" type="button">Lançar na planilha</button>
      <div class="msg" id="msgLanc" role="status" aria-live="polite"></div>
    </section>
    <section class="card">
      <b style="font-size:14px">Meus últimos lançamentos</b>
      <table>
        <thead><tr><th>Data</th><th>Semente</th><th>Nº Ticket</th><th>QTD</th></tr></thead>
        <tbody id="lista"></tbody>
      </table>
      <p class="vazio hide" id="vazio">Nenhum lançamento ainda.</p>
    </section>
  </div>
</main>

<script>
// ===== CONFIGURAÇÃO =====
const URL_SCRIPT = "https://script.google.com/macros/s/AKfycbzxtu2tBxsPVRkW2RlhVAiZ45s2T7eDD6JXc6M9hPmAgbxRloetYucyP95BYLp8bRiE/exec";  // cole aqui a URL do Apps Script (termina em /exec)
const TIMEOUT_MS = 15000;
// ========================

const $ = id => document.getElementById(id);
const FILA_KEY = "sementes_fila";
let sessao = null;
let enviando = false;
const demoUsers = {maria:"1234", joao:"1234"};
const demoRows = [];

/* ---------- utilidades ---------- */
const esc = s => String(s ?? "").replace(/[&<>"']/g, c => ({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"}[c]));
const novoId = () => (crypto.randomUUID ? crypto.randomUUID() : Date.now() + "-" + Math.random().toString(16).slice(2));
const fmtData = d => { const t = new Date(d); return isNaN(t) ? "-" : t.toLocaleString("pt-BR",{day:"2-digit",month:"2-digit",hour:"2-digit",minute:"2-digit"}); };

function msg(el, txt, cls){ el.textContent = txt; el.className = "msg " + (cls||""); }

function lerFila(){
  try{ return JSON.parse(localStorage.getItem(FILA_KEY) || "[]"); }catch(e){ return []; }
}
function salvarFila(f){
  try{ localStorage.setItem(FILA_KEY, JSON.stringify(f)); }catch(e){}
  atualizarPill();
}
function filaDoUsuario(){ return lerFila().filter(i => sessao && i.usuario === sessao.usuario); }

function atualizarPill(){
  const n = filaDoUsuario().length;
  const p = $("pill");
  p.textContent = n ? `${n} pendente${n>1?"s":""}` : "";
  p.classList.toggle("on", n > 0);
}

/* ---------- comunicação ---------- */
async function api(body){
  if(!URL_SCRIPT){
    if(demoUsers[body.usuario] !== body.senha) return {ok:false, erro:"Usuário ou senha inválidos"};
    if(body.acao === "lancar"){
      if(!demoRows.some(r => r.id === body.id))
        demoRows.push({id:body.id, data:body.data || new Date().toISOString(), usuario:body.usuario, semente:body.semente, ticket:body.ticket, quantidade:body.quantidade});
      return {ok:true};
    }
    if(body.acao === "listar") return {ok:true, linhas:demoRows.filter(r=>r.usuario===body.usuario).slice(-8).reverse()};
    return {ok:true, nome:body.usuario};
  }
  const ctl = new AbortController();
  const t = setTimeout(() => ctl.abort(), TIMEOUT_MS);
  try{
    // sem header Content-Type: evita preflight (CORS) no Apps Script
    const r = await fetch(URL_SCRIPT, {method:"POST", body:JSON.stringify(body), signal:ctl.signal});
    if(!r.ok) throw new Error("HTTP " + r.status);
    return await r.json();
  } finally { clearTimeout(t); }
}

/* ---------- fila offline ---------- */
async function enviarFila(){
  if(!sessao) return;
  const todos = lerFila();
  const meus = todos.filter(i => i.usuario === sessao.usuario);
  if(!meus.length) return;
  let restantes = todos.filter(i => i.usuario !== sessao.usuario);
  for(let i = 0; i < meus.length; i++){
    try{
      const r = await api({...sessao, acao:"lancar", id:meus[i].id, data:meus[i].data, semente:meus[i].semente, ticket:meus[i].ticket, quantidade:meus[i].quantidade});
      if(!r.ok){ restantes.push(meus[i]); continue; }   // recusado: mantém para você ver
    }catch(e){
      restantes = restantes.concat(meus.slice(i));       // sem rede: guarda o resto
      break;
    }
  }
  salvarFila(restantes);
}

/* ---------- ações ---------- */
async function entrar(){
  const u = $("usuario").value.trim().toLowerCase(), s = $("senha").value;
  if(!u || !s) return msg($("msgLogin"), "Informe usuário e senha", "erro");
  $("btnEntrar").disabled = true;
  msg($("msgLogin"), "Entrando...");
  try{
    const r = await api({acao:"login", usuario:u, senha:s});
    if(!r.ok) return msg($("msgLogin"), r.erro || "Não foi possível entrar", "erro");
    sessao = {usuario:u, senha:s};
    $("nomeUser").textContent = r.nome || u;
    $("telaLogin").classList.add("hide");
    $("telaLanc").classList.remove("hide");
    $("btnSair").classList.remove("hide");
    $("senha").value = "";
    msg($("msgLogin"), "");
    atualizarPill();
    await enviarFila();
    carregar();
    $("semente").focus();
  }catch(e){
    msg($("msgLogin"), "Sem conexão. Tente de novo.", "erro");
  }finally{ $("btnEntrar").disabled = false; }
}

async function lancar(){
  if(enviando) return;
  const semente = $("semente").value.trim().replace(/\s+/g, " ");
  const ticket = $("ticket").value.trim();
  const quantidade = Number($("qtd").value);
  if(!semente) return msg($("msgLanc"), "Informe o nome da semente", "erro");
  if(!ticket) return msg($("msgLanc"), "Informe o número do ticket", "erro");
  if(!Number.isInteger(quantidade) || quantidade < 1) return msg($("msgLanc"), "A quantidade deve ser um número inteiro maior que zero", "erro");

  const item = {id:novoId(), usuario:sessao.usuario, data:new Date().toISOString(), semente, ticket, quantidade};
  enviando = true; $("btnLancar").disabled = true;
  msg($("msgLanc"), "Enviando...");
  try{
    const r = await api({...sessao, acao:"lancar", id:item.id, data:item.data, semente, ticket, quantidade});
    if(!r.ok) return msg($("msgLanc"), r.erro || "Não foi possível lançar", "erro");
    msg($("msgLanc"), "✔ Lançado com sucesso", "ok");
    limpar();
    enviarFila().then(carregar);
  }catch(e){
    // sem sinal: guarda no aparelho e envia quando der
    salvarFila(lerFila().concat(item));
    msg($("msgLanc"), "Sem conexão. Lançamento guardado, será enviado quando a internet voltar.", "erro");
    limpar();
    carregar();
  }finally{ enviando = false; $("btnLancar").disabled = false; }
}

function limpar(){ $("semente").value = ""; $("ticket").value = ""; $("qtd").value = ""; $("semente").focus(); }

async function carregar(){
  let linhas = [];
  try{
    const r = await api({...sessao, acao:"listar"});
    linhas = r.linhas || [];
  }catch(e){}
  const pend = filaDoUsuario().slice().reverse().map(p => ({...p, pendente:true}));
  const todas = pend.concat(linhas);
  $("lista").innerHTML = todas.map(l =>
    `<tr><td${l.pendente?' class="pend"':''}>${l.pendente ? "⏳ pendente" : fmtData(l.data)}</td><td>${esc(l.semente)}</td><td>${esc(l.ticket)}</td><td>${esc(l.quantidade)}</td></tr>`
  ).join("");
  $("vazio").classList.toggle("hide", todas.length > 0);
}

function sair(){ sessao = null; location.reload(); }

/* ---------- eventos ---------- */
if(!URL_SCRIPT) $("demo").classList.remove("hide");
$("btnEntrar").onclick = entrar;
$("btnLancar").onclick = lancar;
$("btnSair").onclick = sair;
$("senha").addEventListener("keydown", e => { if(e.key === "Enter") entrar(); });
$("usuario").addEventListener("keydown", e => { if(e.key === "Enter") $("senha").focus(); });
$("semente").addEventListener("keydown", e => { if(e.key === "Enter") $("ticket").focus(); });
$("ticket").addEventListener("keydown", e => { if(e.key === "Enter") $("qtd").focus(); });
$("qtd").addEventListener("keydown", e => { if(e.key === "Enter") lancar(); });
window.addEventListener("online", () => enviarFila().then(() => sessao && carregar()));
</script>
</body>
</html>
