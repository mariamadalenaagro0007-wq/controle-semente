const ABA = 'sementes'; // nome da aba dos lançamentos

// usuario: [senha, nome que vai para a coluna Colaborador]
const USUARIOS = {
'maria': ['1234', 'Maria Madalena'],
'joao': ['1234', 'João'],
// adicione uma linha por colaborador
};

function doPost(e) {
const d = JSON.parse(e.postData.contents);
const u = USUARIOS[String(d.usuario).toLowerCase()];
if (!u || u[0] !== String(d.senha)) return out({ok: false, erro: 'Usuário ou senha inválidos'});
const nome = u[1];

if (d.acao === 'login') return out({ok: true, nome});

const ss = SpreadsheetApp.getActive();
const aba = ss.getSheetByName(ABA) || ss.getSheets()[0];

if (d.acao === 'lancar') {
const linha = aba.getRange('C:C').getValues().filter(String).length + 1;
aba.getRange(linha, 1).setValue(new Date()).setNumberFormat('dd/MM/yyyy HH:mm:ss');
aba.getRange(linha, 3).setValue(d.ticket);
aba.getRange(linha, 4).setValue(d.quantidade);
aba.getRange(linha, 6).setValue(nome);
return out({ok: true});
}

if (d.acao === 'listar') {
const n = aba.getLastRow() - 1;
if (n < 1) return out({ok: true, linhas: []});
const linhas = aba.getRange(2, 1, n, 6).getValues()
.filter(r => r[5] === nome && r[2] !== '')
.slice(-8).reverse()
.map(r => ({data: r[0], ticket: r[2], quantidade: r[3]}));
return out({ok: true, linhas});
}
return out({ok: false, erro: 'Ação inválida'});
}

function out(o) {
return ContentService.createTextOutput(JSON.stringify(o))
.setMimeType(ContentService.MimeType.JSON);
}
