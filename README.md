
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Controle de Sementes Comigo</title>
<style>
:root{--verde:#1f6b3a;--verde2:#e8f3ec;--txt:#1d2b22;--borda:#cfdcd3}
*{box-sizing:border-box}
body{margin:0;font-family:system-ui,Arial,sans-serif;background:#f4f7f5;color:var(--txt)}
header{background:var(--verde);color:#fff;padding:16px;display:flex;justify-content:space-between;align-items:center}
header h1{font-size:18px;margin:0}
main{max-width:480px;margin:0 auto;padding:16px}
.card{background:#fff;border:1px solid var(--borda);border-radius:14px;padding:16px;margin-bottom:16px}
label{display:block;font-size:13px;font-weight:600;margin:12px 0 4px}
input,select{width:100%;padding:12px;font-size:16px;border:1px solid var(--borda);border-radius:10px;background:#fff}
button{width:100%;margin-top:16px;padding:14px;font-size:16px;font-weight:700;color:#fff;background:var(--verde);border:0;border-radius:10px}
button.sair{width:auto;margin:0;padding:8px 12px;font-size:13px;background:#ffffff33}
.msg{margin-top:12px;font-size:14px;min-height:18px}
.ok{color:var(--verde)}.erro{color:#b3261e}
table{width:100%;border-collapse:collapse;font-size:13px}
th,td{text-align:left;padding:8px 4px;border-bottom:1px solid var(--borda)}
.demo{background:#fff6d6;border:1px solid #ecd888;border-radius:10px;padding:8px 12px;font-size:12px;margin-bottom:12px}
.hide{display:none}
</style>
</head>
<body>
<header>
  <h1>🌱 Controle de Sementes Comigo</h1>
  <button class="sair hide" id="btnSair">Sair</button>
</header>
<main>
  <div class="demo hide" id="demo">Modo demonstração: nada vai para a planilha ainda. Teste com usuário <b>maria</b> e senha <b>1234</b>.</div>

  <section class="card" id="telaLogin">
    <label>Usuário</label><input id="usuario" autocapitalize="none" autocomplete="username">
    <label>Senha</label><input id="senha" type="password" autocomplete="current-password">
    <button id="btnEntrar">Entrar</button>
    <div class="msg" id="msgLogin"></div>
  </section>

  <div class="hide" id="telaLanc">
    <section class="card">
      <div style="font-size:14px">Olá, <b id="nomeUser"></b></div>
      <label>Número do tíquete</label><input id="ticket" inputmode="numeric">
      <label>Quantidade</label><input id="qtd" type="number" inputmode="decimal" step="any" placeholder="kg ou sacas">
      <label>Tipo de semente</label><select id="tipo"></select>
      <button id="btnLancar">Lançar na planilha</button>
      <div class="msg" id="msgLanc"></div>
    </section>
    <section class="card">
      <b style="font-size:14px">Meus últimos lançamentos</b>
      <table><thead><tr><th>Data</th><th>Tíquete</th><th>Qtd</th><th>Tipo</th></tr></thead><tbody id="lista"></tbody></table>
    </section>
  </div>
</main>

<script>
// ===== CONFIGURAÇÃO =====
const URL_SCRIPT = "";  // cole aqui a URL do Apps Script (termina em /exec)
const TIPOS = ["Soja","Milho","Sorgo","Feijão","Algodão","Trigo","Outro"];
// ========================

const $ = id => document.getElementById(id);
let sessao = null;
const demoUsers = {maria:"1234", joao:"1234"};
const demoRows = [];

async function api(body){
  if(!URL_SCRIPT){
    if(demoUsers[body.usuario] !== body.senha) return {ok:false, erro:"Usuário ou senha inválidos"};
    if(body.acao === "lancar"){ demoRows.push({data:new Date().toISOString(), usuario:body.usuario, ticket:body.ticket, quantidade:body.quantidade, tipo:body.tipo}); return {ok:true}; }
    if(body.acao === "listar") return {ok:true, linhas:demoRows.filter(r=>r.usuario===body.usuario).slice(-8).reverse()};
    return {ok:true, nome:body.usuario};
  }
  const r = await fetch(URL_SCRIPT, {method:"POST", body:JSON.stringify(body)});
  return r.json();
}

function msg(el, txt, cls){ el.textContent = txt; el.className = "msg " + (cls||""); }

async function entrar(){
  msg($("msgLogin"), "Entrando...");
  try{
    const u = $("usuario").value.trim().toLowerCase(), s = $("senha").value;
    const r = await api({acao:"login", usuario:u, senha:s});
    if(!r.ok) return msg($("msgLogin"), r.erro, "erro");
    sessao = {usuario:u, senha:s};
    $("nomeUser").textContent = r.nome || u;
    $("telaLogin").classList.add("hide"); $("telaLanc").classList.remove("hide"); $("btnSair").classList.remove("hide");
    msg($("msgLogin"), ""); carregar();
  }catch(e){ msg($("msgLogin"), "Erro de conexão", "erro"); }
}

async function lancar(){
  const ticket = $("ticket").value.trim(), qtd = $("qtd").value;
  if(!ticket || !qtd) return msg($("msgLanc"), "Preencha tíquete e quantidade", "erro");
  msg($("msgLanc"), "Enviando...");
  try{
    const r = await api({...sessao, acao:"lancar", ticket, quantidade:Number(qtd), tipo:$("tipo").value});
    if(!r.ok) return msg($("msgLanc"), r.erro, "erro");
    msg($("msgLanc"), "✔ Lançado com sucesso", "ok");
    $("ticket").value = ""; $("qtd").value = ""; carregar();
  }catch(e){ msg($("msgLanc"), "Erro de conexão, tente de novo", "erro"); }
}

async function carregar(){
  try{
    const r = await api({...sessao, acao:"listar"});
    $("lista").innerHTML = (r.linhas||[]).map(l =>
      `<tr><td>${new Date(l.data).toLocaleString("pt-BR",{day:"2-digit",month:"2-digit",hour:"2-digit",minute:"2-digit"})}</td><td>${l.ticket}</td><td>${l.quantidade}</td><td>${l.tipo}</td></tr>`).join("");
  }catch(e){}
}

function sair(){ sessao = null; location.reload(); }

$("tipo").innerHTML = TIPOS.map(t => `<option>${t}</option>`).join("");
if(!URL_SCRIPT) $("demo").classList.remove("hide");
$("btnEntrar").onclick = entrar; $("btnLancar").onclick = lancar; $("btnSair").onclick = sair;
$("senha").addEventListener("keydown", e => { if(e.key === "Enter") entrar(); });
</script>
</body>
</html>
